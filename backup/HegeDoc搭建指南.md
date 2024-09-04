## 0. 环境说明
- 阿里云轻量应用服务器
- CentOS 7.6
- 已备案域名一个：xxx.com
- nginx/1.20.1
- Docker version 24.0.2
- 工具
	- xshell
	- xftp
## 1. 使用docker-compose 部署服务

```yaml
version: '3'
services:
  database:
    image: postgres:15.2-alpine
    environment:
      - POSTGRES_USER=hedgedoc
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=hedgedoc
    volumes:
      ["./database:/var/lib/postgresql/data"]
    restart: always
  app:
    # Make sure to use the latest release from https://hedgedoc.org/latest-release
    image: quay.io/hedgedoc/hedgedoc:1.9.6
    environment:
      - CMD_DB_URL=postgres://hedgedoc:password@database:5432/hedgedoc
      - CMD_DOMAIN=123.56.202.155
      - CMD_PROTOCOL_USESSL=true
      - CMD_URL_ADDPORT=false
    volumes:
      ["./uploads:/hedgedoc/public/uploads"]
    ports:
      - "3001:3000"
    restart: always
    depends_on:
      - database
```

### 遇到的问题

![image-20230620205622477](https://images--1.oss-cn-beijing.aliyuncs.com/pic/image-20230620205622477.png)

应该是镜像没拉取到，更换镜像版本解决

![image-20230620205728777](https://images--1.oss-cn-beijing.aliyuncs.com/pic/image-20230620205728777.png)

### 使用Nginx进行反向代理

### 1. 修改环境参数 重新部署docker

![image-20230620211943203](https://images--1.oss-cn-beijing.aliyuncs.com/pic/image-20230620211943203.png)

### 2. 使用Nginx进行部署

- 获取SSL证书（可参看[这里](https://blog.csdn.net/JENREY/article/details/124802587)）

- 修改Nginx配置文件 /etc/nginx/nginx.conf

  ```yaml
  # HegeDocTest
      map $http_upgrade $connection_upgrade {
          default upgrade;
          ''      close;
      }
      server {
          # 服务器端口使用443，开启ssl, 这里ssl就是上面安装的ssl模块
          listen       443 ssl;
          # 域名，多个以空格分开
          server_name  example.com;
  
          # ssl证书地址
          ssl_certificate     /etc/nginx/ssl/hedge.crt;  # pem文件的路径
          ssl_certificate_key  /etc/nginx/ssl/hedge.key; # key文件的路径
  
          # ssl验证相关配置
          ssl_session_timeout  5m;    #缓存有效期
          ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;    #加密算法
          ssl_protocols TLSv1 TLSv1.1 TLSv1.2;    #安全链接可选的加密协议
          ssl_prefer_server_ciphers on;   #使用服务器端的首选算法
          location / {
                  proxy_pass http://127.0.0.1:3001;
                  proxy_set_header Host $host; 
                  proxy_set_header X-Real-IP $remote_addr; 
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
                  proxy_set_header X-Forwarded-Proto $scheme;
          }
          location /.well-known/ {
              root /usr/share/nginx/html/hedge;
          }
          location /socket.io/ {
                  proxy_pass http://127.0.0.1:3001;
                  proxy_set_header Host $host; 
                  proxy_set_header X-Real-IP $remote_addr; 
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
                  proxy_set_header X-Forwarded-Proto $scheme;
                  proxy_set_header Upgrade $http_upgrade;
                  proxy_set_header Connection $connection_upgrade;
          } 
          
      }  
  ```

- 重新启动Nginx : nginx

- 重新加载配置文件：nginx -s reload

- 结束

  ![image-20230620214839156](https://images--1.oss-cn-beijing.aliyuncs.com/pic/image-20230620214839156.png)

<!-- ##{"timestamp":1687510799}## -->