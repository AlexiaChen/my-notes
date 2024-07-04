
## 安装

```bash
sudo apt update && sudo apt install nginx
```


## 配置https SSL

在http block下面加入，这里是比较完成的配置:

```conf
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

        server {
           #SSL 默认访问端口号为 443
           listen 443 ssl;
           #请填写绑定证书的域名
           server_name testapi.gabby.world;
           #请填写证书文件的相对路径或绝对路径
           ssl_certificate /home/ubuntu/Nginx/testapi_gabby_world.pem;
           #请填写私钥文件的相对路径或绝对路径
           ssl_certificate_key /home/ubuntu/Nginx/testapi_gabby_world.key;
           ssl_session_timeout 5m;
           #请按照以下协议配置
           ssl_protocols TLSv1.2 TLSv1.3;
           #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
           ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
           ssl_prefer_server_ciphers on;
           location /api/ {
              proxy_pass http://127.0.0.1:5000/api/;
           }

        }

        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
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
}

```

检查conf文件的语法和当前使用的配置文件地址 `sudo nginx -t`

Dump出当前conf的配置 `sudo nginx -T > nginx.conf`

重启Nginx: `systemctl status nginx`

[SSL 证书 Nginx 服务器 SSL 证书安装部署-证书安装-文档中心-腾讯云 (tencent.com)](https://cloud.tencent.com/document/product/400/35244)

[SSL with NGINX with .pem files - Server Fault](https://serverfault.com/questions/1080193/ssl-with-nginx-with-pem-files)

[Nginx: How do I forward an HTTP request to another port? - Server Fault](https://serverfault.com/questions/536576/nginx-how-do-i-forward-an-http-request-to-another-port)

[Configuring HTTPS servers (nginx.org)](http://nginx.org/en/docs/http/configuring_https_servers.html)

