---

title: "Docker 基础操作、常用命令及错误处理笔记"

date: 2025-10-23 21:00:00 +0800

categories: [工具使用, 学习记录, Docker]

tags: [Docker, 容器, 镜像, Docker命令, 错误处理, 环境部署]

---

## 一、Docker 基础认知

### 1. 核心概念

- 镜像（Image）：Docker 运行的基础模板，包含运行应用所需的代码、环境、依赖等，可理解为“容器的快照”，只读不可修改。

- 容器（Container）：镜像的运行实例，可启动、停止、重启、删除，容器之间相互隔离，拥有独立的文件系统和网络空间。

- 仓库（Repository）：用于存储和分发Docker镜像，分为公有仓库（如Docker Hub）和私有仓库（如公司内部私有仓库）。

-  Docker Compose：用于定义和运行多容器Docker应用的工具，通过yaml文件配置多个容器的依赖关系和运行参数。

## 二、Docker 常见基础操作

### 1. Docker 服务管理（启动/停止/重启）

1.  启动Docker服务：`systemctl start docker`（Linux系统）；Windows/Mac直接启动Docker应用。

2.  停止Docker服务：`systemctl stop docker`。

3.  重启Docker服务：`systemctl restart docker`。

4.  设置Docker开机自启：`systemctl enable docker`。

5.  查看Docker服务状态：`systemctl status docker`。

### 2. 镜像操作（拉取/查看/删除/构建）

1.  拉取镜像（从Docker Hub）：`docker pull 镜像名:标签`（标签不写默认latest，如`docker pull nginx`、`docker pull mysql:8.0`）。

2.  查看本地所有镜像：`docker images`（显示镜像ID、名称、标签、大小）。

3.  查看镜像详情：`docker inspect 镜像ID/镜像名:标签`。

4.  删除本地镜像：`docker rmi 镜像ID/镜像名:标签`（若镜像已创建容器，需先删除容器再删除镜像）；强制删除：`docker rmi -f 镜像ID/镜像名:标签`。

5.  构建本地镜像（通过Dockerfile）：`docker build -t 镜像名:标签 构建目录`（如`docker build -t my-nginx:v1.0 ./`，./为Dockerfile所在目录）。

6.  推送镜像到仓库：`docker push 仓库地址/镜像名:标签`（推送前需先登录仓库，`docker login 仓库地址`）。

### 3. 容器操作（创建/启动/停止/删除/进入）

1.  创建并启动容器：`docker run [参数] 镜像名:标签`（核心参数：-d 后台运行；-p 主机端口:容器端口 端口映射；-v 主机目录:容器目录 目录挂载；--name 容器名 自定义容器名）。

   示例：`docker run -d -p 80:80 --name my-nginx nginx`（后台启动nginx容器，映射主机80端口到容器80端口，容器名my-nginx）。

2.  查看正在运行的容器：`docker ps`；查看所有容器（包括停止的）：`docker ps -a`。

3.  启动已停止的容器：`docker start 容器ID/容器名`。

4.  停止运行中的容器：`docker stop 容器ID/容器名`；强制停止：`docker kill 容器ID/容器名`。

5.  重启容器：`docker restart 容器ID/容器名`。

6.  删除容器：`docker rm 容器ID/容器名`（停止状态下）；强制删除运行中的容器：`docker rm -f 容器ID/容器名`。

7.  进入容器内部（交互式）：`docker exec -it 容器ID/容器名 /bin/bash`（退出容器输入exit，不影响容器运行）。

8.  查看容器日志：`docker logs 容器ID/容器名`；实时查看日志：`docker logs -f 容器ID/容器名`。

9.  查看容器内部进程：`docker top 容器ID/容器名`。

10. 查看容器详情：`docker inspect 容器ID/容器名`。

### 4. Docker Compose 基础操作

1.  编写docker-compose.yml文件（核心配置文件，定义多容器服务），示例（nginx+mysql）：

```
version: '3'
services:
  nginx:
    image: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx/conf:/etc/nginx/conf.d
  mysql:
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=123456
```

2.  启动所有服务：`docker-compose up -d`（在docker-compose.yml所在目录执行）。

3.  停止所有服务：`docker-compose down`（停止并删除容器，不删除镜像）；`docker-compose down -v`（删除挂载目录数据）。

4.  查看服务状态：`docker-compose ps`。

5.  查看单个服务日志：`docker-compose logs -f 服务名`（如`docker-compose logs -f nginx`）。

## 三、Docker 常见错误及处理方法

### 错误1：启动Docker服务报错：Job for docker.service failed because the control process exited with error code.

#### 错误原因：

1.  Docker配置文件（/etc/docker/daemon.json）格式错误；2.  端口被占用；3.  之前Docker服务异常终止，残留进程未清理。

#### 处理方法：

1.  查看错误详情：`journalctl -u docker`，定位具体错误原因。

2.  若为配置文件错误，打开daemon.json文件，修正格式（确保JSON语法正确，逗号、引号无误），修改后重启服务：`systemctl daemon-reload && systemctl restart docker`。

3.  若为端口被占用，使用`netstat -tulpn | grep 端口号`找到占用端口的进程，停止该进程或修改Docker端口映射，再重启Docker。

4.  清理残留进程：`ps -ef | grep docker`，找到残留进程ID，执行`kill -9 进程ID`，再重启Docker。

### 错误2：docker run 报错：port is already allocated

#### 错误原因：

主机端口已被其他容器或进程占用，无法完成端口映射。

#### 处理方法：

1.  查看端口占用情况：`netstat -tulpn | grep 被占用端口`（如`netstat -tulpn | grep 80`）。

2.  方案1：停止占用端口的容器/进程，`docker stop 容器ID`（容器占用）或`kill -9 进程ID`（进程占用），再重新启动当前容器。

3.  方案2：修改端口映射，使用未被占用的主机端口，如将`-p 80:80`改为`-p 8080:80`。

### 错误3：删除镜像报错：Error response from daemon: conflict: unable to delete 镜像ID (must be forced) - image is being used by stopped container 容器ID

#### 错误原因：

该镜像已被一个或多个容器（即使是停止状态）引用，无法直接删除。

#### 处理方法：

1.  先删除引用该镜像的所有容器：`docker rm 容器ID`（多个容器ID用空格分隔）。

2.  再删除镜像：`docker rmi 镜像ID/镜像名:标签`；若仍无法删除，使用强制删除：`docker rmi -f 镜像ID`。

### 错误4：进入容器报错：OCI runtime exec failed: exec failed: unable to start container process: exec: "/bin/bash": stat /bin/bash: no such file or directory: unknown

#### 错误原因：

部分轻量级镜像（如alpine版本）未安装/bin/bash，无法通过bash进入容器。

#### 处理方法：

1.  改用sh进入容器：`docker exec -it 容器ID/容器名 /bin/sh`。

2.  若需使用bash，可在容器内安装bash：`apt update && apt install -y bash`（Debian/Ubuntu镜像）或`yum install -y bash`（CentOS镜像）。

### 错误5：拉取镜像报错：Error response from daemon: Get "https://registry-1.docker.io/v2/": net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)

#### 错误原因：

网络问题，无法连接Docker Hub（国外仓库），导致镜像拉取失败。

#### 处理方法：

1.  检查网络连接，确保主机能正常访问外网。

2.  配置Docker镜像加速器（国内源，如阿里云、网易云），修改/etc/docker/daemon.json文件：

```
{ "registry-mirrors": ["https://docker.mirrors.aliyun.com"] }
```

3.  重新加载配置并重启Docker：`systemctl daemon-reload && systemctl restart docker`，再重新拉取镜像。

### 错误6：Docker Compose 启动报错：ERROR: Couldn't find a package.json file in "/app"

#### 错误原因：

docker-compose.yml文件中配置的目录挂载错误，导致容器无法找到所需文件（如Node.js项目的package.json）。

#### 处理方法：

1.  检查docker-compose.yml中的volumes配置，确保主机目录路径正确（相对路径以docker-compose.yml所在目录为基准）。

2.  确认主机目录下存在对应的文件（如package.json），若不存在，补充文件后重新启动：`docker-compose up -d`。

## 四、补充说明

1.  Docker 镜像加速器配置：除阿里云外，常用国内加速器还有：

- 网易云：https://hub-mirror.c.163.com

- 腾讯云：https://mirror.ccs.tencentyun.com

-  DaoCloud：https://docker.m.daocloud.io

2.  容器数据持久化：通过-v参数挂载主机目录或数据卷（Volume），避免容器删除后数据丢失，常用场景：数据库容器、配置文件挂载。

3.  镜像瘦身技巧：构建镜像时，使用多阶段构建（Multi-stage Build），只保留运行所需的文件，减少镜像体积。

4.  查看Docker版本：`docker --version`；查看Docker Compose版本：`docker-compose --version`。