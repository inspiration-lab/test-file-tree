# 云服务器部署 JavaWeb 环境（基于 docker）

## 1 切换镜像数据源

前往阿里云开通容器镜像服务，进入控制台后进入镜像加速器，按照流程操作即可。操作文档摘录如下，有删减：

## 1.1 Ubuntu/CentOS

（1）安装／升级 Docker 客户端

推荐安装 1.10.0 以上版本的 Docker 客户端。

（2）配置镜像加速器

Docker 客户端版本大于 1.10.0 的用户，可以通过新建或修改 daemon 配置文件 `/etc/docker/daemon.json` 来使用加速器。添加以下内容：

```json
{
  "registry-mirrors": ["https://58rf2fw7wp0u.mirror.aliyuncs.com"]
}
```

**注意：以上地址仅供参考，应依据实际账户提供的为准**

（3）重启服务

```
systemctl daemon-reload
systemctl restart docker
```

## 1.2 Mac/Windows

登录阿里云后，详见：https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors

## 2 JDK 容器

（1）在 https://hub.docker.com/_/amazoncorretto?tab=tags 下载镜像

```
docker pull amazoncorretto:11.0.6
```

（2）Dockerfile 内容如下：

```
FROM amazoncorretto:11.0.6
MAINTAINER clxering <clxering@xx.com>
# 将jar包添加到容器中的目录
ADD demo.jar /usr/src/
# 运行jar包
ENTRYPOINT ["java","-jar","/usr/src/demo.jar"]
```

（3）上传 jar 到 Dockerfile 所在目录，假设为 `/usr/src/tmp`

（4）制作镜像，参数 `-t` 表示不需要缓存；指定镜像名称为 springbootapp

```
docker build -t springbootapp:test .
```

注：末尾有一个点号

以下为成功生成镜像的提示信息：

```
Sending build context to Docker daemon  24.18MB
Step 1/4 : FROM amazoncorretto:11.0.6
 ---> 2acf862bb8fb
Step 2/4 : MAINTAINER clxering <clxering@gmail.com>
 ---> Running in 166531df6339
Removing intermediate container 166531df6339
 ---> a299bd5fb85a
Step 3/4 : ADD demo.jar /usr/src/
 ---> bcddaeffe8ce
Step 4/4 : ENTRYPOINT ["java","-jar","/usr/src/demo.jar"]
 ---> Running in 95f3f84fda5e
Removing intermediate container 95f3f84fda5e
 ---> dcf0f6acf7d8
Successfully built dcf0f6acf7d8
Successfully tagged springbootapp:latest
```

（5）启动镜像并连接到 MySQL

在 application.yml 配置的连接相关内容如下：

```yml
spring:
  profiles:
    active: product
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://mysqlroot:3306/databasename?
    username: xxxx
    password: xxxx
```

则启动容器命令如下：

```
docker run -d -p 7077:7077 --link <MySQL容器别名>:mysqlroot springbootapp:test
```

## 3 MySQL 容器案例

## 3.1 常规配置步骤

## 3.1.1 获取镜像

在 https://hub.docker.com/_/mysql?tab=tags 下载镜像

```
docker pull mysql:8.0.19
```

## 3.1.2 挂载配置文件 my.cnf

配置文件的挂载仅为了便于修改，非必须步骤。

（1）为获取挂载路径，建立一个测试容器

```
docker run --name testmysql \
-p 3307:3306 -e MYSQL_ROOT_PASSWORD=123456 \
-d mysql:8.0.19
```

> 注：用宿主 3307 端口做测试容器的映射

（2）进入测试容器

```
docker exec -it testmysql bash
```

（3）在测试容器中查找配置文件位置。可用 cat 命令依次查看给出的路径是否存在配置文件，最后发现正确路径为 `/etc/mysql/my.cnf`

```
root@f088aa76bc6f:/# mysql --help | grep my.cnf
                      order of preference, my.cnf, $MYSQL_TCP_PORT,
/etc/my.cnf /etc/mysql/my.cnf ~/.my.cnf
```

注：上述 1-3 步骤仅供参考，若有其他方式可获知配置文件路径，可忽略这些步骤。

（4）在宿主上创建配置文件路径 `mkdir -p /usr/src/mysql/conf`

（5）复制测试容器中的配置文件到上述目录。以后如果要改配置，在挂载路径的配置文件上修改即可。

```
docker cp testmysql:/etc/mysql/my.cnf /usr/src/mysql/conf
```

## 3.1.3 挂载数据库数据

容器在关闭或重启后会清空数据，宜用挂载的方式启动 MySQL，保存数据库数据。

（1）查找数据路径。在测试容器执行 `docker inspect testmysql`，在显示内容中找到 `Mount` 属性，其中的 `Destination` 值即为路径

```json
...
"Mounts": [
    {
        "Type": "volume",
        "Name": "20b889c7bf0ee13f654f588e158a840b3feeedac0f35f0d1fca47b4c4617809e",
        "Source": "/var/lib/docker/volumes/20b889c7bf0ee13f654f588e158a840b3feeedac0f35f0d1fca47b4c4617809e/_data",
        "Destination": "/var/lib/mysql",
        "Driver": "local",
        "Mode": "",
        "RW": true,
        "Propagation": ""
    }
],
...
```

（2）在宿主上创建数据件路径 `mkdir -p /usr/src/mysql/data`

注：若删除旧容器，新建容器时要清空原挂载数据目录，除非更换路径，否则新建的容器将无法启动。

（3）启动正式容器并挂载配置文件和数据

```
docker run --name mysqlroot \
-p 3306:3306 -e MYSQL_ROOT_PASSWORD=root \
--restart=always \
--mount type=bind,src=/usr/src/mysql/conf/my.cnf,dst=/etc/mysql/my.cnf \
--mount type=bind,src=/usr/src/mysql/data,dst=/var/lib/mysql \
-d mysql:8.0.19
```

参数释义：

- `--name`，给容器取一个别名为 mysqlroot
- `-p`，映射端口号，将 mysql 的端口号 3306 映射为 3306
- `-e`，MYSQL_ROOT_PASSWORD : 设置 root 密码为 123456
- `--mount`，挂载文件或目录
- `-d`，后台运行容器，并返回容器 ID
- `mysql:8.0.19`，镜像名称
- `--restart=always`，重新启动 docker 后，带该参数的容器将自行启动

注：如果 `--restart=always` 参数位置放在最后，会报错：

```
docker: Error response from daemon: OCI runtime create failed: container_linux.go:349: starting container process caused "exec: \"--restart=always\": executable file not found in $PATH": unknown.
```

**附加内容**：为已经创建的容器添加 `--restart=always` 参数

```
docker container update --restart=always <容器ID>
```

**附加内容**：重启服务器后执行 docker 命令时报错：`Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?`，此时重启 daemon 及 docker 即可，执行如下命令：

```
systemctl daemon-reload
service docker restart
```

执行 `service docker status` 查看状态，输出信息有 `Active: active (running)` 提示则说明成功重启

## 3.2 Docker 创建运行多个 mysql 容器

假设 3306 端口已经运行一个容器，现在使用宿主机的 3307 端口新开一个容器。执行如下命令：

```
docker run --name newone \
-p 3307:3306 -e MYSQL_ROOT_PASSWORD=root \
--restart=always \
-d mysql:8.0.19
```

## 3.3 docker 容器访问宿主机的 MySQL

使用 ifconfig 命令可以看到，有一个 docker0 和 eth0，在 docker 容器中可以通过 eth0 的 IP 地址加上端口号（3306）这样就可以连接宿主机的 MySQL。另外，nginx 可以通过 docker0 的 IP 地址加上构建容器时指定的端口号进行访问容器。

相关文档：https://dev.mysql.com/doc/refman/8.0/en/docker-mysql-getting-started.html

## 4 Nginx 部署

## 4.1 获取镜像

在 https://hub.docker.com/_/nginx?tab=tags 获取稳定版镜像

```
docker pull nginx:stable
```

## 4.2 核心文件挂载

（1）建立测试容器

```
docker run --name nginxtest -d nginx:stable
```

（2）进入测试容器，**根据系统实际情况** 定位核心文件路径

```
docker exec -it nginxtest bash
```

（3）定位配置文件路径

- /etc/nginx/nginx.conf
- /etc/nginx/conf.d/

> 注：使用整个 conf.d 目录可便于增加虚拟主机配置文件

（3）定位首页目录路径

- /usr/share/nginx/html

（4）定位日志文件路径

- /var/log/nginx

（5）在宿主机新建对应目录，并拷贝对应目录

```
docker cp nginxtest:/etc/nginx/nginx.conf /usr/src/nginxdir/
docker cp nginxtest:/etc/nginx/conf.d/default.conf /usr/src/nginxdir/conf.d/
docker cp nginxtest:/usr/share/nginx/html /usr/src/nginxdir/
docker cp nginxtest:/var/log/nginx /usr/src/nginxdir/log/
```

## 4.3 启动容器

```
docker run --name nginx \
-p 80:80 \
--restart=always \
--mount type=bind,src=/usr/src/nginxdir/nginx.conf,dst=/etc/nginx/nginx.conf \
--mount type=bind,src=/usr/src/nginxdir/conf.d/,dst=/etc/nginx/conf.d/ \
--mount type=bind,src=/usr/src/nginxdir/html,dst=/usr/share/nginx/html \
--mount type=bind,src=/usr/src/nginxdir/log/nginx,dst=/var/log/nginx \
-d nginx:stable
```

注：如果 `--restart=always` 参数位置放在最后，会报错：

```
docker: Error response from daemon: OCI runtime create failed: container_linux.go:349: starting container process caused "exec: \"--restart=always\": executable file not found in $PATH": unknown.
```

## 4.4 使用交互模式启动时需要注意的问题

新建 nginx 容器时使用 -it 参数以交互模式启动，并指定执行命令为 /bin/bash，容器新建成功但浏览器连接 80 端口无响应。新建命令如下：

```
docker run --name nginxtest \
-itd \
--restart=always \
--mount type=bind,src=/usr/src/nginxdir/nginx.conf,dst=/etc/nginx/nginx.conf \
--mount type=bind,src=/usr/src/nginxdir/conf.d,dst=/etc/nginx/conf.d \
--mount type=bind,src=/usr/src/nginxdir/log,dst=/var/log \
-p 80:80 \
nginx:stable \
/bin/bash
```

注：应挂载整个配置文件目录，便于添加虚拟主机配置

查看容器状态：

```
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                NAMES
2acea2197fc3        16af99d71a72        "/bin/bash"         7 minutes ago       Up 7 minutes        0.0.0.0:80->80/tcp   nginxtest
```

根本原因是，如果不使用默认启动命令，容器内部没有启动 nginx。进入容器，查看相关进程，没有发现 nginx 进程：

```
root@df2ea7658f01:/# netstat -ntlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
```

注：可能需要安装 netstat 命令，执行 `apt-get install net-tools` 即可

## 解决方案 1

新建命令改为如下形式，使用默认的启动命令 `nginx -g 'daemon off;'`

```
docker run --name nginxtest \
-d \
--restart=always \
--mount type=bind,src=/usr/src/nginxdir/nginx.conf,dst=/etc/nginx/nginx.conf \
--mount type=bind,src=/usr/src/nginxdir/conf.d,dst=/etc/nginx/conf.d \
--mount type=bind,src=/usr/src/nginxdir/log,dst=/var/log \
-p 80:80 \
nginx:stable
```

此时访问 80 端口可进入欢迎页。容器状态：

```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                PORTS                NAMES
9bbbda715013        16af99d71a72        "nginx -g 'daemon of…"   46 seconds ago      Up 45 seconds         0.0.0.0:80->80/tcp   nginxtest
```

## 解决方案 2

手动开启 nginx。进入容器，执行 `/etc/init.d/nginx`

```
root@df2ea7658f01:/# /etc/init.d/nginx
root@df2ea7658f01:/# netstat -ntlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      290/nginx: master p
```

此时再访问 80 端口即可进入欢迎页。

## 5 其他相关命令

- 使服务自动启动，`systemctl enable docker`
- 使服务不自动启动，`systemctl disable docker`
- 检查服务状态（服务详细信息），`systemctl status docker`
- 检查服务状态（仅显示是否 Active)，`systemctl is-active docker`
- 显示所有已启动的服务，`systemctl list-units --type=service`
- 启动服务，`systemctl start docker`
- 停止服务，`systemctl stop docker`
- 重启服务，`systemctl restart docker`
- 查看已下载的镜像，`docker images`，
- 删除指定容器，`docker rm <容器ID>` 或 `docker container rm <容器ID>`，
- 显示当前运行的容器，`dock ps [-a]` 或 `docker container ls [-a]`，添加参数 `-a` 则显示所有容器，包括停止的
- 使用指定 ID 的镜像，以交互模式启动一个容器，在容器内执行 /bin/bash 命令 `docker run -itd <镜像ID> /bin/bash`
- 进入容器，`docker exec -it <容器 ID> bash`，ID 不必全部填写，可辨别即可。如果不指定 /bin/bash，容器运行后会自动停止
- 查看磁盘剩余空间信息，`df -hl`
- 查看容器完整信息，`docker ps --no-trunc` 或 `docker ps -a --no-trunc`
- 使脚本具有执行权限，`chmod +x ./addcontain.sh`
- 复制 build 下所有文件及目录，`cp -r build/. /usr/src/frontapp`，也适用于 scp
- 递归创建目录，`mkdir -p /usr/src/frontapp`，即使上级目录不存在，会按目录层级自动创建目录

## 6 新建环境的 shell 参考

```bash
#!/bin/bash
jdkappid=$1
nginxid=$2

# create nginx dir
mkdir -p /usr/src/jdkapp
mkdir -p /usr/src/frontapp
mkdir -p /usr/src/nginxdir/conf.d/
mkdir -p /usr/src/nginxdir/log/

# create jdkapp
docker run --name jdkapp \
-itd \
--restart=always \
--mount type=bind,src=/usr/src/jdkapp,dst=/usr/src/jdkapp \
$jdkappid \
/bin/bash

# create testnginx
docker run --name testnginx \
-d \
$nginxid

# copy nginx file
docker cp testnginx:/etc/nginx/nginx.conf /usr/src/nginxdir/
docker cp testnginx:/etc/nginx/conf.d/. /usr/src/nginxdir/conf.d/
docker cp testnginx:/var/log/. /usr/src/nginxdir/log/

# delete testnginx
filter=$(docker ps -a | grep testnginx | awk '{print $1}')
# 停止容器并等待命令结束
docker stop $filter
sleep 5s
# 删除容器并等待命令结束
docker rm $filter
sleep 5s

# create nginxfront
docker run --name nginxfront \
-d \
--restart=always \
--mount type=bind,src=/usr/src/frontapp,dst=/usr/src/frontapp \
--mount type=bind,src=/usr/src/nginxdir/nginx.conf,dst=/etc/nginx/nginx.conf \
--mount type=bind,src=/usr/src/nginxdir/conf.d,dst=/etc/nginx/conf.d \
--mount type=bind,src=/usr/src/nginxdir/log,dst=/var/log \
-p 80:80 \
$nginxid

# check
docker ps -a
```
