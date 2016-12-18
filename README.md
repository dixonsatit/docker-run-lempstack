
# create lempstack with  docker run 

[blog](https://blog.sathit.me/%E0%B8%AA%E0%B8%A3%E0%B9%89%E0%B8%B2%E0%B8%87-lemp-stack-php7-fpm-nginx-mariadb-%E0%B8%94%E0%B9%89%E0%B8%A7%E0%B8%A2-docker-run-495ba8a08d49#.mnh272xzb)

```
docker network create  front-nw
```

```
docker run -d --name Mariadb -e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_USER=php -e MYSQL_PASSWORD=1234 -e MYSQL_DATABASE=php --net front-nw -p 3306:3306 --network-alias db --restart always mariadb:10.1.19
```

create Dockerfile

```
FROM php:7.1.0-fpm-alpine

MAINTAINER Sathit Seethaphon <dixonsatit@gmail.com>

ENV TIMEZONE Asia/Bangkok

RUN apk upgrade --update && apk --no-cache add \
    autoconf tzdata file g++ gcc binutils isl libatomic libc-dev musl-dev make re2c libstdc++ libgcc libcurl curl-dev binutils-libs mpc1 mpfr3 gmp libgomp coreutils freetype-dev libjpeg-turbo-dev libltdl libmcrypt-dev libpng-dev openssl-dev libxml2-dev expat-dev \
    && docker-php-ext-install -j$(nproc) iconv mysqli pdo pdo_mysql curl bcmath mcrypt mbstring json xml zip opcache \
	&& docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
	&& docker-php-ext-install -j$(nproc) gd

# TimeZone
RUN cp /usr/share/zoneinfo/${TIMEZONE} /etc/localtime \
&& echo "${TIMEZONE}" >  /etc/timezone

# Install Composer && Assets Plugin
RUN php -r "readfile('https://getcomposer.org/installer');" | php -- --install-dir=/usr/local/bin --filename=composer \
&& composer global require --no-progress "fxp/composer-asset-plugin:~1.2" \
&& apk del tzdata \
&& rm -rf /var/cache/apk/*

EXPOSE 9000

CMD ["php-fpm"]

```

build image
```
docker build -t php7fpm .
```
run php7fpm
```
docker run -d --name php7-fpm -v ${PWD}:/var/www/html/ --expose 9001 --net front-nw --restart always --network-alias phpfpm php7fpm
```

create config nginx `nginx.d/docker.conf`
```

server {
   charset utf-8;
   client_max_body_size 128M;

   listen 80; ## listen for ipv4
   #listen [::]:80 default_server ipv6only=on; ## listen for ipv6

   server_name app-frontend.dev;
   root /var/www/html;
   index index.php;

   access_log  /var/log/nginx/frontend-access.log;
   error_log   /var/log/nginx/frontend-error.log;

   location / {
       # Redirect everything that isn't a real file to index.php
       try_files $uri $uri/ /index.php$is_args$args;
   }

   # uncomment to avoid processing of calls to non-existing static files by Yii
   #location ~ \.(js|css|png|jpg|gif|swf|ico|pdf|mov|fla|zip|rar)$ {
   #    try_files $uri =404;
   #}
   #error_page 404 /404.html;

   location ~ \.php$ {
       include fastcgi_params;
       fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
       fastcgi_pass   phpfpm:9000;
       try_files $uri =404;
   }

   location ~ /\.(ht|svn|git) {
       deny all;
   }
}

```

run ningx
```
docker run -d --name nginx --volumes-from php7-fpm --net front-nw --restart always -p 80:80 -v ${PWD}/nginx.d:/etc/nginx/conf.d nginx:1.10.2-alpine
```


