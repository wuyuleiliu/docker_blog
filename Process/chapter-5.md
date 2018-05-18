# docker_blog

## Docker环境下WordPress博客注意事项

按上篇文章搭建起来简单使用时没什么问题，但是实际使用时还是有些需要注意的


###  启动wordpress镜像（新）

```sh
docker 
run -d 
-e WORDPRESS_DB_HOST=xxx.xxx.xxx.xxx:3306 
-e WORDPRESS_DB_USER=user 
-e WORDPRESS_DB_PASSWORD=user123 
-e WORDPRESS_DB_NAME=wordpress 
--name wordpress_8081 
-v /root/data/wp-content:/var/www/html/wp-content 
-v /root/data/wp-admin:/var/www/html/wp-admin 
-v /root/data/robots.txt:/var/www/html/robots.txt 
--restart=always 
-p 8081:80 
wordpress

docker 
run -d 
-e WORDPRESS_DB_HOST=xxx.xxx.xxx.xxx:3306 
-e WORDPRESS_DB_USER=user 
-e WORDPRESS_DB_PASSWORD=user123 
-e WORDPRESS_DB_NAME=wordpress 
--name wordpress_8082 
-v /root/data/wp-content:/var/www/html/wp-content 
-v /root/data/wp-admin:/var/www/html/wp-admin 
-v /root/data/robots.txt:/var/www/html/robots.txt 
--restart=always 
-p 8082:80 
wordpress
```

我这里将其中的3个目录挂载出来，是因为目前我想在一个物理主机下创建多个服务节点，
比如8081和8082两个节点提供服务，使用nginx时对于其中一个节点的操作不能实时同步到另一个节点，
因为有些东西不是存在数据库，而是在本地，例如主题，我在8081节点下载的主题，在8081的wp-content目录下是存在的，
但是在8082下是不存在的，这样就会出现一些问题，所有我将需要同步的目录（排除数据库中的数据）挂载出来共用，
而且这样也有一个好处，就是我在同一部物理主机上再增加节点，可以用同样的脚本，很方便

但是，这里有个重要的地方注意下：在同一部物理主机下！！！你可以这样操作，如果搭建多物理主机，每个主机下还搭建多个服务，
这样就不对了，这种方式就需要分布式应有的方式来做，按我目前水平的想法，应该是直接全部更新镜像，相当于直接升级镜像，
目前我就不搞这么复杂的（也没第二台机器= =。。。）

> wp-admin 这个管理员后台目录经常挂插件，直接放容器外
> wp-content 这个更新主题，图片等资源，不用多说放外边
> robots.txt Robots协议 这个也不多说，自己搜索下用法为什么加这个文件

###  多节点session共享问题

另外上篇文章中，说了一台物理主机下部署多个节点，后台登录会错乱，这是为什么？

到目前为止，按上边的方式也是不行的（后台登录）

后来我想了半天，找了下资料想起来，在我们登录的时候会创建session与服务保存通信，相当于你获得了权限，
我们登录的时候，创建session实际上从一个节点获得的权限，这个节点也保存了对应的session，
那么你下次在后台修改东西时，访问这个节点，会通过session(服务端)和cookie(客户端)来验证你是不是非法访问

那么，多节点情况下就会出现一种状况：我在A节点是有权限的,然而B节点是不知道我已经获得权限的（没有对应的session），
这里就要说下nginx下session共享的问题，大家自己百度下吧,我也了解的只是皮毛

在nginx`upstream`中添加 `ip_hash;` 就能根据访问者ip固定去请求其中一个节点

可以这么理解，我们开了2个节点A，B，访问者访问我们的网站，nginx反向代理到了A节点，那么这个ip的所有请求都会放到A处理，
这样就不会出现session共享问题了

当然，这样也有问题，在我登录后端之后如果我通过nginx反向代理的docker服务停止之后，登录也就失效了，直接404，
这个共享问题后边解决吧 目前先保持这种 毕竟 访问人也不多 后边我找到别的方式来处理共享问题
