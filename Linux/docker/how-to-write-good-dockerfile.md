# Dockerfile Tips

## 目的

- 更快的构建速度
- 更小的镜像大小
- 更少的镜像分层
- 更好利用镜像缓存
- 更友好的理解
- 更方便的使用

## 建议

- 编写 .dockerignore 文件
- 单个容器只运行一个应用
- 合并多个 RUN 指令
- 删除多余的文件
- 基础镜像不适用 last 标签（待定）
- 选择合适的基础镜像（例如 alpine 版本）
- 设置 WORKDIR 和 CMD
- 使用 ENTRYPOINT （待定）
- 在 entrypoint 脚本中使用 exec
- COPY 和 ADD 优先使用前者
- 合理调整 COPY 和 RUN 的顺序
- 设置默认的环境变量、映射端口和数据卷
- 使用 LABEL 设置元数据
- 添加 HEALTHCHECK

## 示例

```dockerfile
  FROM ubuntu

  ADD . /app

  RUN apt-get update
  RUN apt-get upgrade -y
  RUN apt-get install -y nodejs ssh mysql
  RUN cd /app && npm install

  CMD mysql & sshd & npm start
```

构建镜像：

```shell
  docker build -t test .
```

### 1. 编写 `.dockerignore` 文件

构建镜像时，Docker 需要先准备 context，将所有的文件收集到进程中。默认的 context 是 Dockerfile 所在目录的所有文件，可以通过编写 .dockerignore 文件来忽略到不必要的文件，例如 .git 目录

### 2. 一个容器只运行单个应用

劣势：

- 构建时间更长，镜像大小更大
- 多个应用的日志难处理
- 横向扩展浪费资源
- 僵尸进程

优化示例，删除不必要的包，SSH 用 `docker exec` 代替

```dockerfile
  FROM ubuntu

  ADD . /app

  RUN apt-get update
  RUN apt-get upgrade -y

  RUN apt-get install -y nodejs  # ssh mysql
  RUN cd /app && npm install

  CMD npm start
```

### 3. 合并多个 `RUN` 指令

Docker 镜像是分层的

- Dockerfile 中的每个指令都会创建新的镜像层
- 镜像层将被缓存和复用
- 当 Dockerfile 的指令修改了，复制的文件变化了，或者构建镜像时指定的变量不同了，对应的镜像缓存就会失效
- 某一层的镜像缓存失效，它之后的镜像层缓存都会失效
- 镜像层是不可变的，如果我们在某一层添加一个文件，然后在下一层删除它，则镜像中依然包含该文件（只是这个文件在 Docker 容器中不可见）

Docker 镜像类似洋葱，有很多层。为了修改内层，则需要将外层删掉

优化示例，合并 RUN 指令，删除 `apt-get upgrade`（可选），因为它可能导致每次构建的镜像不确定

```dockerfile
  FROM ubuntu

  ADD . /app

  RUN apt-get update \
      && apt-get install -y nodejs \
      && cd /app \
      && npm install

  CMD npm start
```

记住一点，我们只需要将变化频率一样的指令合并。将 安装 node.js 和安装 npm 模块放在一起，每次修改源代码，都需要重新安装 node.js，这并不合适。因此修改为

```dockerfile
  FROM ubuntu

  RUN apt-get update && apt-get install -y nodejs
  ADD . /app
  RUN cd /app && npm install

  CMD npm start
```

### 4. 基础镜像的标签不使用 `latest`

当镜像更新时， latest 标签会指向不同的镜像。所以审慎使用 latest

优化示例

```dockerfile
  FROM ubuntu:16.04

  RUN apt-get update && apt-get install -y nodejs
  ADD . /app
  RUN cd /app && npm install

  CMD npm start
```

### 5. 删除多余的文件

优化示例

```dockerfile
  FROM ubuntu:16.04

  RUN apt-get update && apt-get install -y nodejs && rm -rf /var/lib/apt/lists/*
  ADD . /app
  RUN cd /app && npm install

  CMD npm start
```

### 6. 选择合适的基础镜像

根据示例需求，我们只是运行 node 程序，不需要使用通用镜像

```dockerfile
  FROM ubuntu:16.04

  # RUN apt-get update && apt-get install -y nodejs && rm -rf /var/lib/apt/lists/*
  ADD . /app
  RUN cd /app && npm install

  CMD npm start
```

选择更小的 alpine 版本的 node 镜像

```dockerfile
  FROM node:7-alpine

  ADD . /app
  RUN cd /app && npm install

  CMD npm start
```

### 7. 设置 `WORKDIR` 和 `CMD`

WORKDIR 指令可以设置默认目录，也就是运行 RUN / CMD / ENTRYPOINT 指令的目录

CMD 指令可以设置容器创建后执行的默认命令，最好将 CMD 指令卸载一个数组中

```dockerfile
  FROM node:7-alpine

  WORKDIR /app
  ADD . /app
  RUN npm install

  CMD ["npm", "start"]
```

### 8. 使用 `ENTRYPOINT` （可选）

ENTRYPOINT 是一个脚本，并且会在容器创建后默认执行，将指定的命令作为其参数

_entrypoint.sh_

```shell
  #!/usr/bin/env sh
  # $0 is a script name,
  # $1, $2, $3 etc are passed arguments
  # $1 is our command
  CMD=$1

  case "$CMD" in
    "dev" )
      npm install
      export NODE_ENV=development
      exec npm run dev
      ;;

    "start" )
      # we can modify files here, using ENV variables passed in
      # "docker create" command. It can't be done during build process.
      echo "db: $DATABASE_ADDRESS" >> /app/config.yml
      export NODE_ENV=production
      exec npm start
      ;;

    * )
      # Run custom command. Thanks to this line we can still use
      # "docker run our_image /bin/bash" and it will work
      exec $CMD ${@:2}
      ;;
  esac
```

示例

```dockerfile
  FROM node:7-alpine

  WORKDIR /app
  ADD . /app
  RUN npm install

  ENTRYPOINT ["./entrypoint.sh"]
  CMD ["start"]
```

可以用如下命令运行该镜像

```shell
  # 运行开发版本
  docker run our-app dev

  # 运行生产版本
  docker run our-app start

  # 运行bash
  docker run -it our-app /bin/bash
```

### 9. 在 `entrypoint` 脚本中使用 `exec`

前文脚本中，使用 `exec` 命令运行 node 引用。不使用 exec 的话，我们不能顺利关闭容器，因为 `SIGTERM` 信号会被 `bash` 脚本进程吞没。exec 命令启动的进程可以取代脚本进程，因此所有的信号都会正常工作

### 10. `COPY` 与 `ADD` 优先使用前者

`COPY` - 将文件拷贝到镜像中

`ADD` - 可用于远程下载与解压

```dockerfile
  FROM node:7-alpine

  WORKDIR /app

  COPY . /app
  RUN npm install

  ENTRYPOINT ["./entrypoint.sh"]
  CMD ["start"]
```

### 11. 合理调整 COPY 和 RUN 的顺序

我们应该把**变化最少的部分放在 `Dockerfile` 的前面**，可以充分利用镜像缓存

优化示例

```dockerfile
  FROM node:7-alpine

  WORKDIR /app
  # package.json 的变化比较少，放在前面
  COPY package.json /app
  RUN npm install

  # 源码经常变化，放在后面
  COPY . /app

  ENTRYPOINT ["./entrypoint.sh"]
  CMD ["start"]
```

### 12. 设置默认的环境变量、映射端口和数据卷

```dockerfile
  FROM node:7-alpine

  # 设置默认环境变量
  ENV PROJECT_DIR=/app

  WORKDIR $PROJECT_DIR

  COPY package.json $PROJECT_DIR
  RUN npm install
  COPY . $PROJECT_DIR

  # 设置默认环境变量
  ENV MEDIA_DIR=/media \
      NODE_ENV=production \
      APP_PORT=3000

  # 设置默认的挂载卷
  VOLUME $MEDIA_DIR
  # 默认暴露端口
  EXPOSE $APP_PORT

  ENTRYPOINT ["./entrypoint.sh"]
  CMD ["start"]
```

> `ENV` 指令指定的环境变量在容器中可以使用，`ARG` 指定构建镜像时的变量

### 13. 使用 `LABEL` 设置元数据

使用 LABEL 指令设置元数据，某些时候，一些外部程序可能会需要

### 14. 添加 `HEALTHCHECK`

启动容器时，指定 `--restart always` 选项，在容器崩溃时，Docker 守护进程 `(docker daemon)` 会重启容器，适合需要长时间运行的容器

但是对于容器陷入异常（死循环，配置错误）导致的不可用。使用 `HEALTHCHECK` 指令让 Docker 周期性的检查容器的健康状况，如果正常放回 0，否则返回 1.

```dockerfile
  FROM node:7-alpine  
  LABEL maintainer "jakub.skalecki@example.com"

  ENV PROJECT_DIR=/app  
  WORKDIR $PROJECT_DIR

  COPY package.json $PROJECT_DIR  
  RUN npm install  
  COPY . $PROJECT_DIR

  ENV MEDIA_DIR=/media \  
      NODE_ENV=production \
      APP_PORT=3000

  VOLUME $MEDIA_DIR  
  EXPOSE $APP_PORT

  # 添加 HEALTHCHECK，当请求失败时 'curl --fail' 会返回非 0 状态
  HEALTHCHECK CMD curl --fail http://localhost:$APP_PORT || exit 1

  ENTRYPOINT ["./entrypoint.sh"]  
  CMD ["start"]
```
