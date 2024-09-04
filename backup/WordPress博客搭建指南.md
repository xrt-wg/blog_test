> 本文只说明具体的配置过程，具体的原理可参考其他文章

## 0. 涉及的原理

> 开始前，先尽可能搞清楚一些原理，可以在之后的实践中debug更有方向感，而是不是一顿狂乱的搜搜搜

- Docker容器技术
- Nginx反向代理
- SSL证书

## 1. 环境说明

- 阿里云轻量应用服务器
- CentOS 7.6
- 已备案域名一个：blog.wgexplorer.com
- nginx/1.20.1
- Docker version 24.0.2
- 工具
	- xshell
	- xftp

## 2. 创建容器

1. 说明

   这里使用docker compose创建容器

2. 配置文件

   ```yaml
   version: "3"
   services:
     mysql:
       image: mysql:5.7
       restart: always
       environment:
        - MYSQL_ROOT_PASSWORD=123456
        - MYSQL_DATABASE=wp_db
        - MYSQL_USER=my_wp_user
        - MYSQL_PASSWORD=123456
       volumes:
         ["./mysql:/var/lib/mysql"]
     web:
       depends_on:
         - mysql
       image: wordpress
       restart: always
       links:
        - mysql
       environment:
        - WORDPRESS_DB_PASSWORD=123456
        - WORDPRESS_DB_HOST=mysql:3306
        - WORDPRESS_DB_USER=my_wp_user
        - WORDPRESS_DB_NAME=wp_db
       ports:
        - "0.0.0.0:8080:80"
       working_dir: /var/www/html
       volumes:
         ["./wordpress:/var/www/html"]
   
   ```
   
3. 文件目录说明

    ![image-20230623120614633](https://images--1.oss-cn-beijing.aliyuncs.com/pic/image-20230623120614633.png)

   - mysql和wordpress分别是配置文件中对应的映射文件路径

     ```yaml
      ["./mysql:/var/lib/mysql"]
     ["./wordpress:/var/www/html"]
     ```

   - ssl是之后暂时用来放SSL证书相关的文件的

4. docker-compose up -d创建并启动容器

   - 在.yml文件所在的路径下使用命令

## 3. 安装

- 访问：服务器IP+8080 （之前.yml文件中配置的映射端口）

  ```yaml
  ports:
       - "0.0.0.0:8080:80"
  ```

  

- 根据提示正常安装即可

  ![img](https://www.ruanyifeng.com/blogimg/asset/2018/bg2018021310.png)

  - 注意：WordPress管理者页面对应的路径是： ip+端口号/admin

## 4. 获取域名的SSL证书

- 使用ACME申请并管理证书

## 5. 使用Nginx反向代理

- 配置nginx.conf文件（文件位置：/etc/nginx/nginx.conf)

  - 注意修改自己的域名，容器的映射端口，还有保存的ssl相关文件的位置

  ```php
   # WordPress
      server {
         listen 80;
         server_name blog.wgexplorer.com;
         rewrite ^(.*) https://$host$1 permanent;
      }
  
      server {
          # 服务器端口使用443，开启ssl, 这里ssl就是上面安装的ssl模块
          listen       443 ssl  http2;
          # 域名，多个以空格分开
          server_name  blog.wgexplorer.com;
  
          # ssl证书地址
          ssl_certificate     /etc/nginx/ssl/blog.crt;  # crt文件的路径
          ssl_certificate_key  /etc/nginx/ssl/blog.key; # key文件的路径
  
          # ssl验证相关配置
          ssl_session_timeout  5m;    #缓存有效期
          ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;    #加密算法
          ssl_protocols TLSv1 TLSv1.1 TLSv1.2;    #安全链接可选的加密协议
          ssl_prefer_server_ciphers off;   #使用服务器端的首选算法
          location / {
                  proxy_pass http://127.0.0.1:8080;
                  proxy_redirect  off;
                  proxy_set_header Host $host;
                  proxy_set_header X-Real-IP $remote_addr;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                  proxy_set_header X-Forwarded-Host $server_name;
                  proxy_set_header X-Forwarded-Proto https;
                  proxy_set_header Upgrade $http_upgrade;
                  proxy_set_header Connection "upgrade";
                  proxy_read_timeout 86400;
                  
          }
          
      }  
  ```

  

- 配置wp-config.php文件 （文件位置，就是在之前映射的那个文件夹下：.../WordPress/wordpress/wp-config.php)

  - 在文件中添加如下代码段即可

  ```php
  /*add_6/22*/
  # 用于控制文件访问权限，适用于上传、更新插件
  define('FS_METHOD', "direct");
  define("FS_CHMOD_DIR", 0777);
  define("FS_CHMOD_FILE", 0777);
   
  # 用于配置ssl
  define( 'FORCE_SSL_ADMIN', true );
   
  if (strpos($_SERVER['HTTP_X_FORWARDED_PROTO'], 'https') !== false){
    $_SERVER['HTTPS'] = 'on';
    $_SERVER['SERVER_PORT'] = 443;
   
  }
   
  if (isset($_SERVER['HTTP_X_FORWARDED_HOST'])) {
    $_SERVER['HTTP_HOST'] = $_SERVER['HTTP_X_FORWARDED_HOST'];
   
  }
  ```

- 重新启动NGINX，并加载配置文件
  - 启动：nginx
  - 重新加载：nginx -s reload

## 6. 主题相关配置

- 自己慢慢摸索就行
- 不要沉迷于各种花里胡哨的主题，更加重要的是内容，主题之类的可以之后慢慢调整

## 7. 遇到的问题

- 重定向次数过多（我似乎是因为站点地址弄错了 多了'/'）
  - [两种方法解决wordpress重定向次数过多too many redirects](https://zhuanlan.zhihu.com/p/441174219)

- wordpress 修改头像
  - [wordpress用户本地上传图片修改头像](https://www.wpzzq.com/871.html#:~:text=%E5%9C%A8wordpress%E7%BD%91%E7%AB%99%E5%90%8E%E5%8F%B0-%E6%8F%92%E4%BB%B6-%E5%AE%89%E8%A3%85%E6%8F%92%E4%BB%B6-%E6%90%9C%E7%B4%A2%E5%AE%89%E8%A3%85%E2%80%9CSimple%20Local,Avatars%E2%80%9D-%E5%90%AF%E7%94%A8%E6%8F%92%E4%BB%B6%E3%80%82%20%E7%84%B6%E5%90%8E%E5%86%8D%E7%94%A8%E6%88%B7%E7%BC%96%E8%BE%91%E9%A1%B5%E9%9D%A2%E5%B0%B1%E5%8F%AF%E4%BB%A5%E9%80%9A%E8%BF%87%E6%9C%AC%E5%9C%B0%E4%B8%8A%E4%BC%A0%E5%9B%BE%E7%89%87%E7%9A%84%E6%96%B9%E5%BC%8F%E4%BF%AE%E6%94%B9%E7%AE%A1%E7%90%86%E5%91%98%E6%88%96%E8%80%85%E7%94%A8%E6%88%B7%E7%9A%84%E5%A4%B4%E5%83%8F%E4%BA%86%E3%80%82%20%E4%BB%A5%E4%B8%8A%E5%B0%B1%E6%98%AF%E5%92%8C%E5%A4%A7%E5%AE%B6%E5%88%86%E4%BA%AB%E7%9A%84%E9%80%9A%E8%BF%87%E5%AE%89%E8%A3%85%E4%BF%AE%E6%94%B9%E5%A4%B4%E5%83%8F%E6%8F%92%E4%BB%B6%E6%9D%A5%E5%AE%9E%E7%8E%B0wordpress%E7%94%A8%E6%88%B7%E6%9C%AC%E5%9C%B0%E4%B8%8A%E4%BC%A0%E5%9B%BE%E7%89%87%E4%BF%AE%E6%94%B9%E5%A4%B4%E5%83%8F%E7%9A%84%E5%8A%9F%E8%83%BD%E7%9A%84%E6%96%B9%E6%B3%95%EF%BC%8C%E5%B8%8C%E6%9C%9B%E5%AF%B9%E5%A4%A7%E5%AE%B6%E6%9C%89%E6%89%80%E5%B8%AE%E5%8A%A9%E3%80%82)
- 安装主题遇到：wordpress 413 Request Entity Too Large
  - [Beginners Guide: How to Install a WordPress Theme](https://www.wpbeginner.com/beginners-guide/how-to-install-a-wordpress-theme/)

- 选主题
  - [Npcink - 共享WordPress主题插件以及使用开发等优秀教程](https://www.npc.ink/)
  - [GitHub - seatonjiang/kratos: 📖 WordPress theme that focus on reading experience](https://github.com/seatonjiang/kratos)
  - [9款 WordPress 最美极简主题推荐 ](https://zhuanlan.zhihu.com/p/37993855)

## 8. 参考

- 配置SSL
  - [使用docker 实现wordpress、nginx配置https](https://blog.csdn.net/chf1142152101/article/details/127532916)
- Docker
  - [Docker 微服务教程 ](https://www.ruanyifeng.com/blog/2018/02/docker-wordpress-tutorial.html)

<!-- ##{"timestamp":1688201999}## -->