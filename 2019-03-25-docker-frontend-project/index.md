## 前言

首先介绍一下我们的项目结构，我们的前后端项目是在一个大文件夹中，文件夹下又分为前
端和后端两个文件夹，打包的过程是：

1. 对前端使用 webpack 编译并存放到文件夹 A
2. 对后端项目使用 babel 编译成 ES5 并存放到文件夹 B
3. 把路径 A 的前端代码拷贝到 B 中

我们可以把这些操作写在 package.json 中，假如命令为 `yarn build`。

## 构建镜像

在把项目交付给客户的时候，我们常常会遇到一些问题，比如客户机器的环境不理想，或者
客户机器不能访问外网所以不能安装 npm 包，这个时候 docker 能让这些问题都变得简单
，我们期望最终的效果是，把整个项目打包为一个镜像，然后通过 docker-compose 启动项
目和外部依赖环境，比如 MongoDB、Redis 等。那我们开始吧：

首先我们要在项目中设置 build 和启动的指令，在 docker 构建和启动的时候会用到，比
如我们设置启动的命令为 `yarn prod-start`；

然后把项目打包成一个镜像，在项目的根目录下创建文件 Dockerfile，文件中的内容如下
：

```dockerfile
# 指定我们要使用的初始化镜像，该镜像意味着已经可以使用 nodejs
FROM node:8.15.1-jessie
# 指定在 docker 中工作的目录
WORKDIR /app
# 将项目根目录下的所有文件拷贝到 docker 的 /app 目录下
ADD . /app
# 在 docker 中的工作目录下执行 yarn build 指令，这个时候我们的项目会执行打包
RUN yarn build
# 对外开放 4000 端口
EXPOSE 4000
```

下一步我们开始构建 docker 镜像，在当前目录下执行以下指令：

```
docker build --tag=gab:1.0 .
```

开始执行这条指令后，docker 会找到当前目录下的 Dockerfile，并按照其中的配置进行构
建。

等待构建完成后，我们再执行 `docker images` 会看到已经生成了一个 REPOSITORY 为
gab，TAG 为 1.0 的镜像。

## 项目运行

我们的项目还需要连接 MongoDB，这又怎么操作呢，这里就轮到 docker-compose 出场了。

docker-compose 是一个定义并且运行多个 docker 容器的工具，并且提供一个独立的和外
界隔离的环境。

在项目根目录下创建文件 docker-compose.yml，内容如下：

```yaml
version: '3'
services:
  mongo:
    image: mongo:3.6.11-stretch
    volumes:
      - /data/gab_db:/data/db
    command: mongod --dbpath /data/db
  gab:
    image: gab:1.1
    depends_on:
      - mongo
    ports:
      - 4000:4000
    command: yarn prod-start
    healthcheck:
      test: ['CMD', 'curl', 'http://127.0.0.1:4000']
      interval: 60s
      timeout: 10s
      retries: 3
    autoheal:
      restart: always
      image: willfarrell/autoheal
      environment:
        - AUTOHEAL_CONTAINER_LABEL=all
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock
```

**version** 制定了 yaml 文件格式对应的 docker 版本。

**services** 下定义了项目需要启动的 docker 服务，值得注意的是 **volumes** 字段的
含义是把服务器上的某个目录挂载到本地，格式为：本地目录:服务器目录。

gab 下的 **depends_on** 表示 gab 服务依赖 mongo 服务，因此启动的时候会先启动
mongo 服务，然后再启动 gab。

**command** 字段表示启动 docker 服务的时候执行的命令。

**healthcheck** 定义健康检查：

- **test** 是健康检查执行的命令；
- **interval** 是每隔多少时间进行一次检查；
- **timeout** 是执行这个指令后，多少时间没有得到响应则认为检查失败；
- **retries** 的意思是失败多少次后会对该服务的健康状态标记为不健康。

**autoheal** 的作用是自动重启不健康的 docker 服务，设置
**AUTOHEAL_CONTAINER_LABEL** 为 true 会监听所有的运行中的容器。

## 导入导出

如果需要把项目部署到一个不通外网的环境中，无法通过 docker registry 下载镜像，我
们可以把镜像导出为 tar 包：

```bash
docker save -o gab1.1.tar gab:1.1
```

在需要部署的服务器中，导入镜像：

```bash
docker load --input gab1.1.tar
```
