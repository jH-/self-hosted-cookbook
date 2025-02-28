# Lychee
## Pros
- Very nice and polished UI
- Supports multiple users
- Supports U2F
- Can import from local server path, URL or Dropbox
- Supports creating and sharing nested albums!
- Actively developed
- TIFF support via imagick

## Cons
- No face or object recognition
- No processing for extra image formats
- Built as a gallery hub for arts & graphics, rather than a archiving solution with support for lossless formats

---
- [Github repo](https://github.com/LycheeOrg/Lychee)
- [Homepage](https://lycheeorg.github.io/)
- [Docs](https://lycheeorg.github.io/docs/docker.html)

## docker-compose.yml
```yml
---
services:
#  lychee_cache:
#    image: redis:alpine
#    container_name: lychee_redis
#    hostname: lychee_redis
#    security_opt:
#      - no-new-privileges:true
#    healthcheck:
#      test: ["CMD-SHELL", "redis-cli ping || exit 1"]
#    ports:
#      - ${REDIS_PORT:-6379}:${REDIS_PORT:-6379}
#    user: 1026:100
#    env_file:
#      - path: ./.env
#        required: false
#    environment:
#      - TZ=${TIMEZONE:-UTC}
#    networks:
#      - lychee
#    volumes:
#      - cache:/data:rw
#    restart: on-failure:5

  lychee_db:
    container_name: lychee_db
    image: mariadb:10
    security_opt:
      - no-new-privileges:true
    env_file:
      - path: ./.env
        required: false
    environment:
      - MYSQL_ROOT_PASSWORD=lychee123
      - MYSQL_DATABASE=lychee
      - MYSQL_USER=lychee
      - MYSQL_PASSWORD=lychee123
    expose:
      - 3306
    volumes:
      - mysql:/var/lib/mysql
    networks:
      - lychee
    restart: unless-stopped

  lychee:
    image: lycheeorg/lychee
    container_name: lychee
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped
    depends_on:
      - lychee_db
    ports:
      - 3123:80
    volumes:
      - ./lychee/conf:/conf
      - ./lychee/uploads:/uploads
      - ./lychee/sym:/sym
      - ./lychee/logs:/logs
      - ./lychee/tmp:/lychee-tmp
      #- ./nginx.conf:/etc/nginx/nginx.conf
    env_file:
      - path: ./.env
        required: false
    environment:
      - PUID=${PUID:-1000}
      - PGID=${PGID:-1000}
      - PHP_TZ=Europe/Oslo
      - TIMEZONE=Europe/Oslo
      - DB_CONNECTION=mysql
      - DB_HOST=lychee_db
      - DB_PORT=3306
      - DB_DATABASE=lychee
      - DB_USERNAME=lychee
      - DB_PASSWORD=lychee123
      - STARTUP_DELAY=30
      #- APP_NAME=Laravel
      #- APP_ENV=local
      #- APP_URL=http://localhost #DONT FORGET if hosting on a server
      #- CACHE_DRIVER=file
      #- SESSION_DRIVER=file
      #- SESSION_LIFETIME=120
      #- QUEUE_DRIVER=sync
      #- REDIS_HOST=127.0.0.1
      #- REDIS_PASSWORD=null
      #- REDIS_PORT=6379
      #- MAIL_DRIVER=smtp
      #- MAIL_HOST=smtp.mailtrap.io
      #- MAIL_PORT=2525
      #- MAIL_USERNAME=null
      #- MAIL_PASSWORD=null
      #- MAIL_ENCRYPTION=null

networks:
  lychee:

volumes:
  mysql:
    name: lychee_prod_mysql
    driver: local
#  cache:
#    name: lychee_prod_redis
#    driver: local
```

**Note:** when you open the url first time (e.g. http://localhost:3123) you will see an installer. There should be no need to change anything there, and just clicking "Next" should work.

## nginx.conf
<s>This file is required to increase the upload size (from the default 20MB).
Original can be found here: [default.conf](https://github.com/LycheeOrg/Lychee-Docker/blob/master/default.conf).
This will increase the size to 1000MB (`upload_max_filesize` and `post_max_size`):</s>

```nginx
user www-data;
worker_processes auto;
daemon off;

error_log /var/log/nginx/error.log;
error_log stderr;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    # Maps to exclude successful Docker health checks from stdout
    map $remote_addr $loggable_ip {
        127.0.0.1 "";
        default 1;
    }
    map $status $loggable_status {
        200 "";
        default 1;
    }

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    access_log  /dev/stdout  main if=$loggable_status$loggable_ip;

    sendfile        on;
    keepalive_timeout  65;

    # By default, if the processing of images takes more than 60s,
    # a 504 Gateway timeout occurs, so we increase the timeout here
    # to allow procesing of large images or when multiple images are
    # being processed at the same time. We set max_execution_time
    # below to the same value.
    fastcgi_read_timeout 3600;

    # We also set the send timeout since this can otherwise also cause
    # issues with slow connections
    fastcgi_send_timeout 3600;

    gzip  on;

    server {
        root /var/www/html/Lychee/public;
        listen       80;
        server_name  localhost;
        client_max_body_size 100M;
        
        # serve static files directly
        location ~* \.(jpg|jpeg|gif|css|png|js|ico|html)$ {
            access_log off;
            expires max;
            log_not_found off;
        }

        # removes trailing slashes (prevents SEO duplicate content issues)
        if (!-d $request_filename)
        {
            rewrite ^/(.+)/$ /$1 permanent;
        }

        # If the request is not for a valid file (image, js, css, etc.), send to bootstrap
        if (!-e $request_filename)
        {
            rewrite ^/(.*)$ /index.php?/$1 last;
            break;
        }

        location / {
            index  index.php
            try_files $uri $uri/ /index.php?$query_string;
        }

        # Serve /index.php through PHP
        location = /index.php {
            fastcgi_split_path_info ^(.+?\.php)(/.*)$;

            try_files $uri $document_root$fastcgi_script_name =404;

            # Mitigate https://httpoxy.org/ vulnerabilities
            fastcgi_param HTTP_PROXY "";

            fastcgi_pass unix:/run/php/php8.4-fpm.sock;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PHP_VALUE "post_max_size=100M
                max_execution_time=3600
                upload_max_filesize=100M
                memory_limit=256M";
            fastcgi_param PATH /usr/local/bin:/usr/bin:/bin;
            include fastcgi_params;
        }

        # Deny access to other .php files, rather than exposing their contents
        location ~ [^/]\.php(/|$) {
            return 403;
        }
    }

    include /etc/nginx/conf.d/*.conf;
}
```