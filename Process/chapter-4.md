# docker_blog

## Docker环境下WordPress博客的搭建

> docker pull docker.io/wordpress

没加版本号，根据自己需要来搞吧

Mysql数据库和之前基本一致，数据库db换了下名字而已

nginx.conf 配置文件不需要改动

###  启动wordpress镜像

```sh
docker run 
-d 
-e WORDPRESS_DB_HOST=xxx.xxx.xxx.xxx:3306 
-e WORDPRESS_DB_USER=wordpress 
-e WORDPRESS_DB_PASSWORD=wordpress 
-e WORDPRESS_DB_NAME=wordpress 
--name wordpress_test 
--restart=always 
-p 9091:80 
wordpress
```

查看下日志没有错误，访问正常即可

单节点正常，多节点有点问题，后台登录不了，我慢慢查下原因吧