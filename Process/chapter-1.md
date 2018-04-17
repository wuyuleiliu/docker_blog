# docker_ghost_blog

    
## Mysql数据库的搭建

### 准备工作

docker环境搭建 Docker 软件包和依赖包已经包含在默认的 CentOS-Extras 软件源里

```sh
yum -y install docker

service docker start
```

> docker pull mysql

这里没加版本号，我看了下运行时的版本号是`5.7.21`，不添加版本号默认最新版本，本地lastest

###  启动mysql

```sh
docker run -d \
-e MYSQL_ROOT_PASSWORD=root \
-e MYSQL_USER=user \
-e MYSQL_PASSWORD=user \
-e MYSQL_DATABASE=ghost \
-p 3306:3306 \
-v /var/mysql/data:/var/lib/mysql \
--restart=always \
--name ghost_db mysql \
```

几个参数不想说太多，另外如果想保存数据一定要将数据挂载到服务器上的目录,端口映射主机端口，目前不会搞太复杂的
