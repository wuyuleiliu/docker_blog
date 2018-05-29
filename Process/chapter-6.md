# docker_blog

## Docker环境下更改镜像的时区设置

在`pull`镜像之后我们使用镜像生成容器，监控对应的容器日志，发现大多数都是0区的时间，那么如何改成东八区呢？


###  镜像时区设置

推荐直接在已有的镜像基础之上创建新的镜像，直接使用新镜像就好


```sh
mkdir zonetimesetcreate
cd zonetimesetcreate
touch Dockerfile
vim Dockerfile
```

在Dockerfile文件中添加如下（以nginx为例）：

```sh
FROM docker.io/nginx:latest

RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
  && echo 'Asia/Shanghai' >/etc/timezone \
```

执行命令：

```sh
docker build -t docker.io/nginx:1.0 .
```

创建了一个`docker.io/nginx:1.0`的镜像，直接使用这个新的就好，查看日志，变成东八区了