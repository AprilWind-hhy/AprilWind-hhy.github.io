---

title: "解决阿里云中dockerhu被墙pull不下来的问题"

date: 2026-02-17 21:00:00 +0800

categories: [阿里云, 学习记录, 项目部署]

tags: [云服务器,阿里云]

---

# 一、Docker 环境准备

## 1.1 卸载旧版 Docker 组件

卸载所有已安装的 Docker 组件，避免潜在的冲突和兼容性问题（忽略“软件包未安装”的提示）：

```
# 删除Docker相关源

sudo rm -f /etc/apt/sources.list.d/*docker*.list

# 卸载Docker和相关的软件包

for pkg in docker.io docker-buildx-plugin docker-ce-cli docker-ce-rootless-extras docker-compose-plugin docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove -y $pkg; done

注意：卸载 Docker 不会自动移除镜像、容器、存储卷和网络，这些数据默认存储在 /var/lib/docker/ 目录，需手动删除。
```

## 1.2 安装 Docker 社区版

适配说明

- 阿里云服务器：使用 http://mirrors.cloud.aliyuncs.com

- 非阿里云服务器：替换为 https://mirrors.aliyun.com

```
# 更新包管理工具
sudo apt-get update

# 添加Docker软件包源依赖
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common

# 添加Docker GPG密钥
sudo curl -fsSL http://mirrors.cloud.aliyuncs.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

# 添加Docker软件源
sudo add-apt-repository -y "deb [arch=$(dpkg --print-architecture)] http://mirrors.cloud.aliyuncs.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

# 安装Docker核心组件（含Compose插件）
sudo apt-get -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

替代方案解决。

警告信息：Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).

替代方案（彻底解决警告，推荐）：

```Bash
# 1. 下载GPG密钥并保存到trusted.gpg.d目录（替代apt-key add）
sudo curl -fsSL http://mirrors.cloud.aliyuncs.com/docker-ce/linux/ubuntu/gpg -o /etc/apt/trusted.gpg.d/docker.gpg
# 2. 后续添加软件源、安装Docker步骤不变
```

## 1.3 启动并设置 Docker 开机自启

```
# 检查 Docker 版本（验证安装）
docker --version

# 启动Docker服务
sudo systemctl start docker

# 设置开机自启
sudo systemctl enable docker
```

## 1.4 配置 Docker 镜像加速器（解决拉取镜像失败）

由于运营商网络限制，直接从 Docker Hub 拉取镜像可能失败，需配置国内镜像加速器：

注意事项

docker search 命令会直接查询 Docker Hub，配置加速器后该命令仍可能失败，属正常现象。

配置步骤

```shell
# 1. 停止 Docker 服务（先停掉再改配置）
sudo systemctl stop docker

# 2. 创建/覆盖 daemon.json 配置文件（使用阿里云镜像源）
sudo tee /etc/docker/daemon.json <<-'EOF'
{
    "registry-mirrors": ["https://mirrors.aliyun.com"],
    "insecure-registries": ["mirrors.cloud.aliyuncs.com"]
}
EOF

# 3. 重启 Docker 服务（加载新配置）
sudo systemctl daemon-reload
sudo systemctl start docker

# 4. 验证镜像源是否生效
docker info | grep -A 5 "Registry Mirrors"
```



# 终极兜底方案（完全绕开网络 / DNS）

# 完整操作步骤：导入镜像与项目打包（2023-10-01）

### 完整操作步骤（替换为「本地下载镜像包 → 上传服务器 → 导入构建」）

#### 【前置说明】

所有「本地操作」在你的电脑（Windows/Mac/Linux）终端/命令提示符执行；「服务器操作」在阿里云服务器终端执行；

核心逻辑：先解决 `openjdk:8-jdk-alpine` 镜像拉取问题，再按你的原有流程执行打包、构建、启动。

### 第一步：本地操作 - 下载 OpenJDK 镜像并保存为 tar 包

#### 1.1 本地安装 Docker（若未安装）

- Windows/Mac：下载 [Docker Desktop](https://www.docker.com/products/docker-desktop/)，安装后启动（右下角/docker图标显示「Running」）；
- Linux：本地终端执行 `sudo apt install docker.io -y && sudo systemctl start docker`。

#### 1.2 本地拉取 OpenJDK 镜像并保存为 tar 包

打开本地终端/命令提示符，执行以下命令（**全程在本地执行**）：

```Bash
# 1. 拉取 openjdk:8-jdk-alpine 镜像（本地联网即可）
docker pull openjdk:8-jdk-alpine

# 2. 将镜像保存为 tar 包（保存到本地桌面，方便查找）
# Windows 命令（替换为你的用户名）：
# docker save -o C:\Users\你的用户名\Desktop\openjdk8-jdk-alpine.tar openjdk:8-jdk-alpine

# Mac/Linux 命令：
docker save -o ~/Desktop/openjdk8-jdk-alpine.tar openjdk:8-jdk-alpine
```

执行后，本地桌面会生成 `openjdk8-jdk-alpine.tar` 文件（约 80MB）。

### 第二步：服务器操作 - 准备目录 + 上传文件

#### 2.1 服务器创建目录（全程在服务器终端执行）

```Bash
# 1. 登录服务器后，进入根目录（确保权限）
cd /

# 2. 创建 code 目录（若已存在则跳过）
sudo mkdir -p /code

# 3. 创建项目子目录（和你的项目名一致）
sudo mkdir -p /code/yuoj-code-sandbox-master

# 4. 赋予 code 目录读写权限（避免后续上传/打包报错）
sudo chmod -R 777 /code
```

#### 2.2 上传文件到服务器（3类文件需上传）

##### ① 上传本地项目文件到服务器 `/code/yuoj-code-sandbox-master/`

- 方式1：scp 命令（本地执行）
  - ```Bash
    # 本地终端执行（替换「你的服务器IP」「本地项目路径」）
    scp -r /本地项目路径/yuoj-code-sandbox-master/* root@你的服务器IP:/code/yuoj-code-sandbox-master/
    ```
- 方式2：可视化工具（WinSCP/FileZilla）
  - 连接服务器：IP=你的阿里云服务器IP，端口=22，用户名=root，密码=服务器密码；
  - 本地找到项目文件夹 `yuoj-code-sandbox-master`，将所有文件拖到服务器 `/code/yuoj-code-sandbox-master/` 目录。

##### ② 上传本地的 `openjdk8-jdk-alpine.tar` 到服务器 `/code/` 目录

- 方式1：scp 命令（本地执行）
  - ```Bash
    # Windows（替换用户名和服务器IP）：
    # scp C:\Users\你的用户名\Desktop\openjdk8-jdk-alpine.tar root@你的服务器IP:/code/
    
    # Mac/Linux（本地执行）：
    scp ~/Desktop/openjdk8-jdk-alpine.tar root@你的服务器IP:/code/
    ```
- 方式2：可视化工具：将本地桌面的 `openjdk8-jdk-alpine.tar` 拖到服务器 `/code/` 目录。

##### ③ 上传 Dockerfile 和 docker-compose.service.yml 到对应目录

- 按你的原有要求：
  - `Dockerfile` 上传到服务器 `/code/yuoj-code-sandbox-master/` 目录；
  - `docker-compose.service.yml` 上传到服务器 `/code/` 目录。

### 第三步：服务器操作 - 导入镜像 + 项目打包

#### 3.1 服务器导入 OpenJDK 镜像（解决镜像拉取问题）

在服务器终端执行（**全程在服务器执行**）：

```Bash
# 1. 进入 /code 目录
cd /code

# 2. 导入本地镜像包（无需联网，直接加载）
sudo docker load -i openjdk8-jdk-alpine.tar

# 3. 验证镜像导入成功（能看到 openjdk:8-jdk-alpine 即可）
sudo docker images | grep openjdk
# 预期输出：openjdk  8-jdk-alpine  xxxxxx  About an hour ago
```

#### 3.2 服务器打包项目（生成 target 目录和 jar 包）

```Bash
# 1. 进入项目目录
cd /code/yuoj-code-sandbox-master

# 2. 执行 mvn 打包（若服务器未装 maven，先安装）
# 安装 maven（若未安装）：
# sudo apt install maven -y

# 打包命令（跳过测试，加快速度）
mvn clean package -DskipTests

# 【备用方案】若服务器打包失败：
# 1. 本地执行 mvn clean package -DskipTests，生成 target 目录；
# 2. 将本地 target 目录上传到服务器 /code/yuoj-code-sandbox-master/ 目录；
# 3. 服务器执行：sudo chmod 777 /code/yuoj-code-sandbox-master/target/yuoj-code-sandbox-0.0.1-SNAPSHOT.jar
```

### 第四步：服务器操作 - 构建 + 启动容器（按你的原有流程）

#### 4.1 确认文件路径正确（服务器执行）

```Bash
# 检查 Dockerfile 路径
ls /code/yuoj-code-sandbox-master/Dockerfile

# 检查 docker-compose.service.yml 路径
ls /code/docker-compose.service.yml

# 检查 jar 包路径
ls /code/yuoj-code-sandbox-master/target/yuoj-code-sandbox-0.0.1-SNAPSHOT.jar
```

✅ 所有命令都能输出文件名，说明路径正确。

#### 4.2 执行 docker compose 构建并启动（服务器执行）

```Bash
# 进入 /code 目录
cd /code

# 执行构建+启动命令（你的原有命令，增加 sudo 避免权限问题）
sudo docker compose -f docker-compose.service.yml up yuoj-code-sandbox-master -d
```

#### 4.3 检查容器启动状态（服务器执行）

```Bash
# 查看容器是否启动成功
sudo docker ps -a | grep yuoj-code-sandbox-master

# 若容器未启动（状态为 Exited），查看日志排查问题：
sudo docker logs yuoj-code-sandbox-master
```

### 关键补充说明

1. **Dockerfile 无需修改**：你的原有 Dockerfile 中 `FROM openjdk:8-jdk-alpine` 可直接使用（因为已本地导入该镜像）；
2. **docker-compose.service.yml 无需修改**：保持你原有内容即可；
3. **常见问题排查**：
   1. 若 `docker logs` 显示「jar 包找不到」：检查 `target/yuoj-code-sandbox-0.0.1-SNAPSHOT.jar` 路径是否正确；
   2. 若容器启动后端口不通：检查服务器安全组/防火墙是否开放 8090 端口；
   3. 若权限报错：执行 `sudo chmod -R 777 /code` 赋予全权限。

### 总结

| 操作阶段       | 执行位置 | 核心命令                                                     |
| :------------- | :------- | :----------------------------------------------------------- |
| 本地下载镜像   | 本地     | `docker pull openjdk:8-jdk-alpine && docker save -o 桌面/xxx.tar 镜像名` |
| 服务器导入镜像 | 服务器   | `cd /code && sudo docker load -i openjdk8-jdk-alpine.tar`    |
| 项目打包       | 服务器   | `cd /code/yuoj-code-sandbox-master && mvn clean package -DskipTests` |
| 构建启动容器   | 服务器   | `cd /code && sudo docker compose -f docker-compose.service.yml up -d` |
| 验证启动状态   | 服务器   | `sudo docker ps -a                                           |

按以上步骤执行，可彻底解决镜像拉取问题，完全复用你的原有项目结构和启动流程。