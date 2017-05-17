```sh
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
	worker_connections 768;
	# multi_accept on;
}

http {

	##
	# Basic Settings
	##

	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	keepalive_timeout 65;
	types_hash_max_size 2048;
	# server_tokens off;

	# server_names_hash_bucket_size 64;
	# server_name_in_redirect off;

	include /etc/nginx/mime.types;
	default_type application/octet-stream;

	##
	# SSL Settings
	##

	ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
	ssl_prefer_server_ciphers on;

	##
	# Logging Settings
	##

	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;

	##
	# Gzip Settings
	##

	gzip on;
	gzip_disable "msie6";

	# gzip_vary on;
	# gzip_proxied any;
	# gzip_comp_level 6;
	# gzip_buffers 16 8k;
	# gzip_http_version 1.1;
	# gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

	##
	# Virtual Host Configs
	##

	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;

    ## 配置需要代理的server地址，vertx是为该代理server起的名字
    ## weight代表该server的权重
	upstream vertx {
		keepalive 100;
    	server      127.0.0.1:8080;
        server      127.0.0.1:8081 weight=2;
    }

    ## proxy_pass: 地址一定要喝upstream中的名字一致
    server{
        listen      8081;
        location / {
	        root   html;
	        index  index.html index.htm;
	        proxy_pass http://vertx;      # 负载均衡指向的发布服务tomcat
	        proxy_http_version 1.1;
			proxy_set_header Connection "";
	    }
    }
}

```
