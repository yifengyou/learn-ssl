# nginx双向验证

```
server {
     listen       443;    //默认端口
     server_name  localhost;

     ssl on;
     ssl_certificate      /etc/nginx/ssl/server.crt;
     ssl_certificate_key  /etc/nginx/ssl/server.key;
     
     ssl_verify_client on;
     ssl_client_certificate /etc/nginx/ca/private/ca.crt;

     ssl_session_timeout 5m;
     ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
     ssl_ciphers ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
     ssl_prefer_server_ciphers on;

    location / {
        proxy_pass        http://server_pool;
        proxy_set_header   Host             $host;
        proxy_set_header   X-Real-IP        $remote_addr;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        root   /usr/share/nginx/html;
        index  index.html index.htm;
   }
}
```
