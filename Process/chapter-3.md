# docker_blog

## Docker环境下Nginx的搭建

> docker pull nginx

没加版本号，根据自己需要来搞吧

创建nginx配置文件目录（自己随意，docker容器中的nginx配置文件挂载在外部服务器上）

```sh
cd root
mkdir nginx
cd nginx
```

在这个配置文件将nginx.conf写好，下面是个示例:

```sh
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    upstream myServer{
		#内网ip 不开外网端口
        server xxx.xxx.xxx.xxx:8081;
        server xxx.xxx.xxx.xxx:8082;
    }
	
	server {
        listen       80;
		#外网访问ip
        server_name  xxx:xxx:xxx:xxx;

        location / {
            
            proxy_pass http://myServer/;
			proxy_set_header Host $host;
            proxy_set_header X-Real-Ip $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

    }
}
```

---

###  启动nginx

```sh
docker run -p 80:80 --name mynginx -v /root/nginx/nginx.conf:/etc/nginx/nginx.conf -d docker.io/nginx
```

查看下日志没有错误，访问正常即可

### nginx问题

nginx访问日志中出现其他网站的GET记录

```sh
xxx.xxx.xxx.xxx - - [24/Apr/2018:11:10:27 +0000] "GET http://www.baidu.com/cache/global/img/gs.gif" 301 0 "-" "-" "-"
xxx.xxx.xxx.xxx - - [24/Apr/2018:11:10:32 +0000] "GET http://www.baidu.com/cache/global/img/gs.gif" 301 0 "-" "-" "-"
xxx.xxx.xxx.xxx - - [24/Apr/2018:11:16:55 +0000] "http://www.baidu.com/cache/global/img/gs.gif" 400 174 "-" "-" "-"
xxx.xxx.xxx.xxx - - [24/Apr/2018:11:17:23 +0000] "GET http://www.baidu.com/cache/global/img/gs.gif" 301 0 "-" "-" "-"
xxx.xxx.xxx.xxx - - [24/Apr/2018:11:18:31 +0000] "http://www.baidu.com/" 400 174 "-" "-" "-"
xxx.xxx.xxx.xxx - - [24/Apr/2018:11:18:42 +0000] "GET http://www.baidu.com/" 200 8856 "-" "-" "-"
xxx.xxx.xxx.xxx - - [24/Apr/2018:11:19:35 +0000] "GET http://www.baidu.com/ HTTP/1.1" 200 8645 "-" "-" "-"
xxx.xxx.xxx.xxx - - [24/Apr/2018:11:22:02 +0000] "GET http://www.baidu.com/ HTTP/1.1" 200 8856 "-" "-" "-"
```

比如上边跳转到百度首页，居然返回200，那是否成功了呢？自己的服务器是否被当做代理访问其他网站了呢？

200状态仅仅说明Apach/Nginx正确发送了数据，并不在意这些数据从哪来的。

事实上，RFC2616规范中说明了，Apache就是要接受这些请求的，这意味这即使proxying关闭了，Apache也要收纳这些个看起来像是代理的请求。
但是Apache没有了代理，所以不会去看目标网站，只是把自己的内容返回过去，一般来说，就是这个默认的页面。


这边用 telnet 来测试下

```sh
[root@gc-server ~]# telnet xxx.xxx.xxx.xxx  80
Trying xxx.xxx.xxx.xxx...
Connected to xxx.xxx.xxx.xxx.
Escape character is '^]'.
GET http://www.baidu.com/ HTTP/1.1
Host: www.baidu.com
```

回车两次，查看是否是自己的网站页面，我试了下 还是自己主页，所以没问题

```sh
xxx.xxx.xxx.xxx - - [24/Apr/2018:11:31:46 +0000] "CONNECT  http://www.baidu.com/ HTTP/1.1" 404 170 "-" "-" "-"
```

这个CONNECT方法，也是Proxy的一种，一般是用来做SSL管道代理的，当然端口可以有很多种可能。

一般来说，如果禁用了Proxy，这样的请求Apache会回复405状态码的（Method not allowed）。
虽然不会有什么问题，不过PHP还是要运行一下尝试做这个事情，所以我们最好直接禁用除了我们想要的方法（一般就是GET，POST，HEAD）之外的所有方法。
最好的方法嘛，我们除了自己的域名，其他的莫名其妙的请求都屏蔽掉！

```sh
if ($host !~ ^(example.com|www.example.com)$) {
	return 444;
}
```

### 增加ip访问限制

将原来的容器缩掉，重新创建新容器，准备好对应的目录和文件

>  配置文件   /root/nginx/nginxconfig/nginx.conf       
>  文件目录   /root/nginx/nginxconfig/conf.d   
>  日志路径   /root/nginx/log/

nginx.conf 配置文件内容：


```sh
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
   
    #禁止访问ip
    include /etc/nginx/conf.d/blockip.conf;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    upstream myServer{
		#内网ip 不开外网端口
        server xxx.xxx.xxx.xxx:8081;
        server xxx.xxx.xxx.xxx:8082;
    }
	
	server {
        listen       80;
        server_name  xxx.xxx.xxx.xxx;
	
        if ($http_user_agent ~* (Wget|ab) ) { return 403; } 
        
        if ($http_user_agent ~* LWP::Simple|BBBike|wget) { return 403; }	
        
        if ($host !~ ^(xxx.xxx.xxx.xxx)$) {
            return 444;
        }
	
        location / {
            
            proxy_pass http://myServer/;
			proxy_set_header Host $host;
            proxy_set_header X-Real-Ip $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

    }
}
```

conf.d 下创建 default.conf，blockip.conf

default.conf 直接默认的nginx文件就好,这里也贴下吧

```sh
server {
    listen       80;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```

blockip.conf 存放黑名单ip(xxx.xxx.xxx.xxx 代表 ip)

```sh
deny xxx.xxx.xxx.xxx;
deny xxx.xxx.xxx.xxx;
```

查看ip访问量

```sh
awk '{print $1}' /root/nginx/log/access.log |sort |uniq -c|sort -n
```

启动容器

```sh
docker run 
-p 80:80 
--name mynginx 
-v /root/nginx/nginxconfig/nginx.conf:/etc/nginx/nginx.conf 
-v /root/nginx/nginxconfig/conf.d:/etc/nginx/conf.d 
-v /root/nginx/log/:/var/log/nginx/ 
-d docker.io/nginx
```