# Dockerfile

## 基本格式

```dockerfile
  # Comment
  INSTRUCTION arguments
```

Notes:

* 顺序执行指令
* 第一条指令必须是 `FROM`
* *以 `#` 开头的行表示注释*，其他位置会被当成参数
* 指令`（INSTRUCTION）`不区分大小写，但为了与参数区分，*推荐使用大写*
* 错误的指令会被忽略

## 指令`（INSTRUCTIONS）`

### FROM

格式：`FROM <image> or FROM <image>:<tag>`

### ENV

格式：`ENV <key> <value> or ENV <key>=<value>`

可以申明容器的环境变量，可以被特定指令（ENV、ADD、COPY、WORKDIR、EXPOSE、VOLUME、USER）直接使用

其他指令使用环境变量时，需使用 `$variable_name` 或 `${variable_name}`

ONBUILD 不支持使用

### COPY

格式：`COPY <src> <dest>`

Notes:

* `<src>` 源可以有多个文件，但必须是`上下文根目录`的相对路径
* `<src>` 支持通配符，如 `COPY test* /dir/` 表示拷贝以 `test*` 开头的文件
* `<dest>` 可以是文件或目录，是*目标镜像的绝对路径*或*相对于 WORKDIR 的路径*
* `<dest>` 如果目录不存在，会被自动创建

### ADD

格式：`ADD <src> <dest>`

Notes：

* `<src>` 可以指向网络文件 URL
* `<dest>` 指向目录，则 URL 必须是完整路径，这样可以获得文件名 filename，该文件会被复制到 `<dest>/<filename>`
* `<src>` 指向本地归档文件，会在复制到容器时被解压提取
* `<src>` 指向网络中的归档文件，则不会被解压提取

### EXPOSE

格式：`EXPOSE <port> [<port>/<protocol>...]`

声明容器打算使用哪个端口、哪个协议（默认是 TCP），并不会进行端口映射

```dockerfile
  EXPOSE 80 === EXPOSE 80/tcp
  EXPOSE 80/udp
```

### USER

格式：`USER <user>[:group] or USER <UID>[:GID]`

`USER` 指令设置 user name 和 user group（optional）

在它之后的 RUN、CMD 和 ENTRYPOINT 指令会以设置的 user 来执行

### WORKDIR

格式：`WORKDIR /path/to/workdir`

Notes：

* 设置目录后，之后的 RUN、CMD、ENTRYPOINT、COPY 和 ADD 都会在此目录下运行
* 若目录不存在，则会自动创建
* 可多次使用该命令
* 若是相对路径，则是相对于上一个 WORKDIR 设置的路径

```dockerfile
  WORKDIR /a
  WORKDIR b
  WORKDIR c
  RUN pwd
  # output /a/b/c
```

### RUN

格式：`RUN <command>` (shell) or `RUN ["executable", "param1", "param2"]` (exec)

`RUN` 指令会在前一个镜像上创建一个容器运行命令，在结束后提交容器为新的镜像

使用 shell 格式时，命令通过 `/bin/sh -c` 运行

使用 exec 格式时，命令是直接运行的，不调用 shell 程序。
> exec 格式的参数会被当成 JSON 数据解析，故需使用双引号，环境变量参数也不会被替换

### CMD

格式：`CMD <command>`（shell）

格式：`CMD ["executable", "param1", "param2"]`（exec）

格式：`CMD ["param1", "param2"]`（为 ENTRYPOINT 提供参数）

Notes：

* 指定容器运行时的默认值，可以是一条指令，也可以是一些参数
* 多条 CMD 指令，只有最后一条有效
* 如果运行容器时 `docker run` 指Y定命令，则会被覆盖

### ENTRYPOINT

格式：`ENTRYPOINT <command>`（shell）

格式：`ENTRYPOINT ["executable", "param1", "param2"]`（exec）

Notes：

* 使用 shell格式时，忽略任何的 CMD 和 docker run 参数
* 使用 shell格式，使用 /bin/sh -c 运行，是其子进程，进程在容器中的 PID 不是 1，docker stop 将不起效
* 使用 exec 格式时，与 shell 格式效果相反
* ENTRYPOINT 指令可以被追加参数，但不能被覆盖，与 CMD 指令不同

```dockerfile
  FROM busybox
  WORKDIR /app
  COPY run.sh /app
  RUN chmod +x run.sh
  ENTRYPOINT ["/app/run.sh"]
  CMD ["param1"]
```