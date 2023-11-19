# 简单使用 docker 配置开发环境（mac+php）


<!--more-->

#### 前言
- 搭建开发环境是一个开发者必备的技能，但是重复的搭建各种开发环境，依然是一件很无趣且浪费时间的事情。
- 以前的解决方案通常是虚拟机或者一些集成工具，它们能很好的解决重复搭建相同开发环境的问题。
- 同时它们也有着一些缺陷，比如虚拟机过于笨重，集成工具功能比较固定单一。
- docker 等虚拟容器技术的出现，让开始构建多样的开发环境成为可能。


#### 基础需求
- 安装 [docker](https://www.docker.com/) 
- 安装 [docker-compose](https://docs.docker.com/compose/) 
- windows 环境：需要win10 + wsl 子系统 ，参考 [WIN10 系统下 WSL 配置 laravel 开发环境指南#](https://learnku.com/articles/22842)

#### 第一步 clone docker-compose 脚本,复制配置文件

```shell
# 克隆脚本
git clone https://github.com/snowlyg/dnmp
#复制配置文件
copy docker-compose.sample.yml docker-compose.yml
```

#### 根据自己需要的服务，修改配置文件解开或者加上对应的注释代码
- php 基础开发环境nginx + mysql + redis 如下

```yml
version: "3"
services:
  nginx:
    build:
      context: ./services/nginx
      args:
        NGINX_VERSION: 1.15.7-alpine
        CONTAINER_PACKAGE_URL: mirrors.aliyun.com
        NGINX_INSTALL_APPS: ""
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./www:/www/:rw
      - ./services/nginx/ssl:/ssl:rw
      - ./services/nginx/conf.d:/etc/nginx/conf.d/:rw
      - ./services/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./services/nginx/fastcgi-php.conf:/etc/nginx/fastcgi-php.conf:ro
      - ./services/nginx/fastcgi_params:/etc/nginx/fastcgi_params:ro
      - ./logs/nginx:/var/log/nginx/:rw
    environment:
      TZ: "Asia/Shanghai"
    restart: always
    networks:
      - default

  php:
    build:
      context: ./services/php
      args:
        PHP_VERSION: php:7.3-fpm-alpine
        CONTAINER_PACKAGE_URL: mirrors.aliyun.com
        PHP_EXTENSIONS: "pdo_mysql,mysqli,mbstring,gd,curl,opcache,soap,amqp,bcmath,pcntl,redis,xdebug"
        TZ: "Asia/Shanghai"
    container_name: php
    expose:
      - 9501
    extra_hosts:
      - "www.site1.com:172.17.0.1"
    volumes:
      - ./www:/www/:rw
      - ./services/php/php.ini:/usr/local/etc/php/php.ini:ro
      - ./services/php/php-fpm.conf:/usr/local/etc/php-fpm.d/www.conf:rw
      - ./logs/php:/var/log/php
      - ./data/composer:/tmp/composer
    restart: always
    cap_add:
      - SYS_PTRACE
    networks:
      - default

#  php56:
#    build:
#      context: ./services/php
#      args:
#        PHP_VERSION: php:5.6.40-fpm-alpine
#        CONTAINER_PACKAGE_URL: mirrors.aliyun.com
#        PHP_EXTENSIONS: "pdo_mysql,mysqli,mbstring,gd,curl,opcache"
#        TZ: "Asia/Shanghai"
#    container_name: php56
#    expose:
#      - 9501
#    volumes:
#      - ./www:/www/:rw
#      - ./services/php/php.ini:/usr/local/etc/php/php.ini:ro
#      - ./services/php/php-fpm.conf:/usr/local/etc/php-fpm.d/www.conf:rw
#      - ./logs/php:/var/log/php
#      - ./data/composer:/tmp/composer
#    restart: always
#    cap_add:
#      - SYS_PTRACE
#    networks:
#      - default

  #  php54:
  #    build:
  #      context: ./services/php54
  #      args:
  #        PHP_VERSION: php:5.4.45-fpm
  #        CONTAINER_PACKAGE_URL: mirrors.aliyun.com
  #        PHP_EXTENSIONS: "pdo_mysql,mysqli,mbstring,gd,curl,opcache"
  #        TZ: "Asia/Shanghai"
  #    container_name: php54
  #    volumes:
  #      - ./www:/www/:rw
  #      - ./services/php54/php.ini:/usr/local/etc/php/php.ini:ro
  #      - ./services/php54/php-fpm.conf:/usr/local/etc/php-fpm.d/www.conf:rw
  #      - ./logs/php:/var/log/php
  #      - ./data/composer:/tmp/composer
  #    restart: always
  #    cap_add:
  #      - SYS_PTRACE
  #    networks:
  #      - default

  mysql:
   image: mysql:8.0.13
   container_name: mysql
   ports:
     - "3306:3306"
   volumes:
     - ./services/mysql/mysql.cnf:/etc/mysql/conf.d/mysql.cnf:ro
     - ./data/mysql:/var/lib/mysql/:rw
   restart: always
   networks:
     - default
   environment:
     MYSQL_ROOT_PASSWORD: "123456"
     TZ: "Asia/Shanghai"

  # mysql5:
  #   image: mysql:5.7.28
  #   container_name: mysql5
  #   ports:
  #     - "3305:3306"
  #   volumes:
  #     - ./services/mysql5/mysql.cnf:/etc/mysql/conf.d/mysql.cnf:ro
  #     - ./data/mysql5:/var/lib/mysql/:rw
  #   restart: always
  #   networks:
  #     - default
  #   environment:
  #     MYSQL_ROOT_PASSWORD: "123456"
  #     TZ: "Asia/Shanghai"

  #  openresty:
  #    image:  openresty/openresty:alpine
  #    container_name: openresty
  #    ports:
  #       - "80:80"
  #       - "443:443"
  #    volumes:
  #       - ./www:/www/:rw
  #       - ./services/openresty/conf.d:/etc/nginx/conf.d/:ro
  #       - ./services/openresty/openresty.conf:/ssl:rw
  #       - ./services/openresty/fastcgi-php.conf:/usr/local/openresty/nginx/conf/nginx.conf:ro
  #       - ./services/openresty/fastcgi_params:/usr/local/openresty/nginx/conf/fastcgi-php.conf:ro
  #       - ./services/openresty/ssl:/usr/local/openresty/nginx/conf/fastcgi_params:ro
  #       - ./logs/nginx:/var/log/nginx/:rw
  #    environment:
  #      TZ: "Asia/Shanghai"
  #    networks:
  #      - default

  redis:
    image: redis:5.0.3-alpine
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - ./services/redis/redis.conf:/etc/redis.conf:ro
      - ./data/redis:/data/:rw
    restart: always
    entrypoint: ["redis-server", "/etc/redis.conf"]
    environment:
      TZ: "Asia/Shanghai"
    networks:
      - default

  #  memcached:
  #    image: memcached:alpine
  #    container_name: memcached
  #    ports:
  #      - "11211:11211"
  #    environment:
  #       MEMCACHED_CACHE_SIZE: "128"
  #    networks:
  #      - default

  # rabbitmq:
  #   image: rabbitmq:management
  #   container_name: rabbitmq
  #   restart: always
  #   ports:
  #     - "5672:5672"
  #     - "15672:15672"
  #   environment:
  #      TZ: "Asia/Shanghai"
  #      RABBITMQ_DEFAULT_USER: "myuser"
  #      RABBITMQ_DEFAULT_PASS: "mypass"
  #   networks:
  #         - default

  # phpmyadmin:
  #  image: phpmyadmin/phpmyadmin:latest
  #  container_name: phpmyadmin
  #  ports:
  #    - "8080:80"
  #  volumes:
  #    - ./services/phpmyadmin/config.user.inc.php:/etc/phpmyadmin/config.user.inc.php:ro
  #    - ./services/phpmyadmin/php-phpmyadmin.ini:/usr/local/etc/php/conf.d/php-phpmyadmin.ini:ro
  #  networks:
  #    - default
  #  environment:
  #    - PMA_HOST=mysql
  #    - PMA_PORT=3306
  #    - TZ=Asia/Shanghai

  # phpredisadmin:
  #  image: erikdubbelboer/phpredisadmin:latest
  #  container_name: phpredisadmin
  #  ports:
  #    - "${REDISMYADMIN_HOST_PORT}:80"
  #  networks:
  #    - default
  #  environment:
  #    - REDIS_1_HOST=redis
  #    - REDIS_1_PORT=6379
  #    - TZ=Asia/Shanghai

  # mongodb:
  #  image: mongo:4.1
  #  container_name: mongodb
  #  environment:
  #      MONGO_INITDB_ROOT_USERNAME: "root"
  #      MONGO_INITDB_ROOT_PASSWORD: "123456"
  #      TZ: "Asia/Shanghai"
  #  volumes:
  #    - ./data/mongo:/data/db:rw
  #    - ./data/mongo_key:/mongo:rw
  #  ports:
  #     - "27017:27017"
  #  networks:
  #     - default
  #  command:
  #     --auth

  #  adminmongo:
  #    image: mrvautin/adminmongo
  #    container_name: adminmongo
  #    ports:
  #      - "1234:1234"
  #    environment:
  #      - HOST=0.0.0.0
  #      - DB_HOST=mongodb
  #      - DB_PORT=27017
  #    networks:
  #      - default

  # elasticsearch:
  #  build:
  #    context: ./services/elasticsearch
  #    args:
  #      ELASTICSEARCH_VERSION: 7.1.1
  #      ELASTICSEARCH_PLUGINS: ""
  #  container_name: elasticsearch
  #  environment:
  #    - TZ=Asia/Shanghai
  #    - discovery.type=single-node
  #    - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
  #  volumes:
  #    - ./data/esdata:/usr/share/elasticsearch/data
  #    - /services/elasticsearch/elasticsearch.yml:/usr/share/elasticsearch/elasticsearch.yml
  #  hostname: elasticsearch
  #  restart: always
  #  ports:
  #    - "9200:9200"
  #    - "9300:9300"

#  kibana:
#    image: kibana:7.1.1
#    container_name: kibana
#    environment:
#      TZ: "Asia/Shanghai"
#      elasticsearch.hosts: http://elasticsearch:9200
#      I18N_LOCALE: "zh-CN"
#    hostname: kibana
#    depends_on:
#      - elasticsearch
#    restart: always
#    ports:
#      - "5601:5601"

#  logstash:
#    image: logstash:7.1.1
#    container_name: logstash
#    hostname: logstash
#    restart: always
#    depends_on:
#      - elasticsearch
#    environment:
#      TZ: "Asia/Shanghai"
#    ports:
#      - "9600:9600"
#      - "5044:5044"


networks:
  default:
```

#### 在脚本根目录执行 docker-compose 命令，构建环境
- 如果本机安装了对应的服务，比如 apache,nginx,mysql 等，会导致对应80，3306 等端口别占用。
- 需要修改前面的配置的端口映射关系项 
```yml
# 修改为，前面的端口是本机端口，后面的是容器内的端口。
# 这个设置表示将 本机端口映射到容器的80端口
# 后面访问的时候也需要使用 83 端口，比如 http://localhost:83s
ports:
  - "83:80" 


``` 
- ，
```shell
  # 构建本启动脚本未注释部分的服务，并在守护对应服务
  docker-compose up -d
  # 如果只启动部分服务，比如 php
  docker-compose up -d php
  # 或者 php mysql
  docker-compose up -d php mysql
```

#### 查看对应服务是否正常启动
- 可以直接在面板看
![apps](./apps.jpg)

- 或者使用命令行 `docker ps` 查看
 ![apps-cli](./apps-cli.png) 

- 可以看到对应服务的名称，状态，创建时间和对应的端口

#### 部署一个应用到这个 docker 搭建的 php 环境中
```shell
#进入 nginx 配置目录
cd services/nginx/conf.d/
# 新建一个配置文件 loclhost.conf
server {
  listen 80;
  server_name localhost;
  root /www;

  #add_header X-Frame-Options "SAMEORIGIN";
  #add_header X-XSS-Protection "1; mode=block";
  #add_header X-Content-Type-Options "nosniff";

  index index.html index.htm index.php;

  charset utf-8;

  location / {
    #try_files $uri $uri/ /index.php?$query_string;
    if (!-e $request_filename) {
      rewrite  ^(.*)$  /index.php?s=/$1  last;
    }
  }

  location = /favicon.ico { access_log off; log_not_found off; }
  location = /robots.txt  { access_log off; log_not_found off; }

  error_page 404 /index.php;

  location ~ \.php$ {
    fastcgi_pass  php:9000;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
    include fastcgi_params;
  }

  location ~ /\.(?!well-known).* {
    deny all;
  }
}
# 注意 location ~\.php$ {} 里面， fastcgi_pass 的值是 php:9000 不是通常的 127.0.0.1，至于原理后面会提到

# 新建一个 index.html 文件到 www/ 目录下,再访问 http://localhost 就能看到 index.html 里面的内容了。 
```

#### 容易踩坑点

- 部署正式项目的时候，配置 mysql ,redis 的服务地址配置的时候不能使用常规的  localhost 或者 127.0.0.1。
- 因为 php,mysql,redis,nginx 都部署再相对隔离的不同 docker 容器中，就如同把它们安装再了不同机器上一样。
- 要在容器中访问其他容器 ，需要使用它们的容器名称，也是它们的网络地址别名，比如 mysql,php 等等。
- 所以前面配置 nginx 设置的时候 location ~\.php$ {} 里面， fastcgi_pass 的值是 php:9000 不是通常的 127.0.0.1

