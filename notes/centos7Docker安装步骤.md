# centos7 Docker安装与部署

## 前言

公司由于项目的原因，需要在一个全新的服务器上运行项目。按照我们目前的部署模式使用了CI/CD，将java后台代码直接打包成了镜像。我们不想改变这种打包模式，从而选择了在新的服务器上也安装docker，本项目按照官方文档一步一步的进行安装。

## 1. Docker Engine安装

 	服务器版本，我们这里使用VirtualBox构建的虚拟机来部署，linux版本选择的是Centos 7.9。部署步骤，我们按照[官网](https://docs.docker.com/engine/install/)上的步骤一步一步走就行了。

1.  卸载老版本，目前我们重新装的系统，所有这种情况是不存在的

```
 sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

2. 安装存储库，存储库提供安装和更新Docker

```
sudo yum install -y yum-utils

sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

3. 安装*最新版本*的 Docker Engine 和 containerd

   ```
   sudo yum install docker-ce docker-ce-cli containerd.io
   ```

4. 启动Docker

```
sudo systemctl start docker
```

5. run hello-world测试

```
sudo docker run hello-world
```

6. 配置 Docker 开机启动

   设置开机启动

   ```
    sudo systemctl enable docker.service
    sudo systemctl enable containerd.service
   ```

   禁用开机启动

   ```
   sudo systemctl disable docker.service
   sudo systemctl disable containerd.service
   ```

## 2. Docker Compose安装

​	Compose 是一个用于定义和运行多容器 Docker 应用程序的工具。借助 Compose，您可以使用 YAML 文件来配置应用程序的服务。然后，使用单个命令，从配置中创建并启动所有服务。

1. 运行此命令以下载 Docker Compose 的当前稳定版本：

   ```
   sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   
   ```

2. 对二进制文件应用可执行权限

   ```
   sudo chmod +x /usr/local/bin/docker-compose
   ```

3. 测试安装

   ```
   docker-compose --version
   ```

## 3. 构建桥接网络

​	我们通过 `docker network create xxx` 命令构建一个我们需要的网络，为什么要自己构建呢？

​	因为我们部署的容器不只是我们的应用程序，我们应用程序还需要依赖mysql，redis这种服务才能运行。所以我们需要让这些服务连到同一个网络才能让这些服务相互访问。

​	虽然docker 默认提供了一个network，我们可以通过 `docker network ls` 查看确实有一个名称为`bridge`的网络，那为什么我们不直接用这个网络，还要自己创建，这个的话官网上有相关说明

​	**用户定义的网桥在容器之间提供自动 DNS 解析**。

默认桥接网络上的容器只能通过 IP 地址相互访问，除非您使用`--link`选项，这被认为是遗留的。在用户定义的桥接网络上，容器可以通过名称或别名相互解析。

## 4. 构建Docker Compose文件

构建Mysql文件

```
#依从的 compose 版本
version: "2"
services:
  #构建db服务
  db:
    #容器名称，在同一个桥接网络host使用 db 来访问mysql
    container_name: db
    #自动重启
    restart: always
    #镜像名称
    image: mysql:5.7.32
    #添加环境变量设置初始密码
    environment:
      - MYSQL_ROOT_PASSWORD=123456
      - LANG=C.UTF-8
    #添加本地的地址存储mysql的数据（持久化数据）
    volumes:
      - ./volume/db:/var/lib/mysql
    #配置网络
    networks: 
      - xxx
networks:
  xxx:
    external: true
```

## 5. 构建应用

例如我使用的是Springboot开发的java应用

1. 在yaml配置文件中，我们将修改以下配置

   `url: jdbc:mysql://db:3306/forestry?serverTimezone=Asia/Shanghai&characterEncoding=utf8&useSSL=false`

2. 配置Dockerfile
3. 配置docker compose

## 6. 运行docker compose文件

将生成好的docker compose文件放到服务器上，使用命令  `docker-compose up -d` 分别运行mysql和应用程序的文件