# docker_ghost_blog

## Docker环境下Ghost博客的搭建

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