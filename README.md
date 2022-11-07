# FalconPress

Deploy WordPress with the highest performance options currently possible, in Docker-Compose.

Still under heavy construction.

## Redis - Object Cache

## MariaDB - Database
```
wordpress-db:
    image: mariadb:latest
    restart: unless-stopped
    env_file:
      - .db.env
    volumes:
      - "/var/lib/mysql/data:${MARIADB_DATA_DIR}"
      - "/var/lib/mysql/logs:${MARIADB_LOG_DIR}"
      - /var/docker/mariadb/conf:/etc/mysql
    environment:
        MYSQL_ROOT_PASSWORD: "${MYSQL_ROOT_PASSWORD}"
        MYSQL_DATABASE: "${MYSQL_DATABASE}"
        MYSQL_USER: "${MYSQL_USER}"
        MYSQL_PASSWORD: "${MYSQL_PASSWORD}"
```


## Wordpress-FPM
```
  wordpress:
    image: wordpress:php8.1-fpm-alpine
    volumes:
      - /data/html:/var/www/html
    depends_on:
      - mysql
    environment:
      WORDPRESS_DB_HOST: wordpress-db
      MYSQL_ROOT_PASSWORD: "${MYSQL_ROOT_PASSWORD}"
      WORDPRESS_DB_NAME: "${MYSQL_DATABASE}"
      WORDPRESS_DB_USER: "${MYSQL_USER}"
      WORDPRESS_DB_PASSWORD: "${MYSQL_PASSWORD}"
      WORDPRESS_TABLE_PREFIX: wp_
    links:
      - wordpress-db
    restart: unless-stopped
```

### Install
- 


## Nginx
```
  nginx:
    image: nginx:latest
    volumes:
      - ./nginx:/etc/nginx/conf.d
      - ./logs/nginx:/var/log/nginx
      - ./data/html:/var/www/html
    ports:
      - 8080:80
    links:
      - wordpress
```

### nginx.conf
```
fastcgi_cache_path /etc/nginx/cache levels=1:2 keys_zone=WORDPRESS:100m inactive=60m;
fastcgi_cache_key "$scheme$request_method$host$request_uri";
server {
    listen 80

    access_log   /var/log/nginx/example.com.access.log;
    error_log    /var/log/nginx/example.com.error.log;

    root /var/www/html;
    index index.php;

    server_name unixcat.de;

    set $skip_cache 0;

    # POST requests and urls with a query string should always go to PHP
    if ($request_method = POST) {
        set $skip_cache 1;
    }   
    if ($query_string != "") {
        set $skip_cache 1;
    }   

    # Don't cache uris containing the following segments
    if ($request_uri ~* "/wp-admin/|/xmlrpc.php|wp-.*.php|/feed/|index.php|sitemap(_index)?.xml") {
        set $skip_cache 1;
    }   

    # Don't use the cache for logged in users or recent commenters
    if ($http_cookie ~* "comment_author|wordpress_[a-f0-9]+|wp-postpass|wordpress_no_cache|wordpress_logged_in|PHPSESSID") {
        set $skip_cache 1;
    }

    location / {
        try_files $uri $uri/ /index.php?$args;
    }    

    location ~ \.php$ {
        try_files $uri =404; 
        include fastcgi_params;
        fastcgi_pass wordpress:9000;
        fastcgi_index index.php;

        fastcgi_cache_bypass $skip_cache;
            fastcgi_no_cache $skip_cache;

        fastcgi_cache WORDPRESS;
        fastcgi_cache_valid 200 60m;
    }

    location ~ /purge(/.*) {
        fastcgi_cache_purge WORDPRESS "$scheme$request_method$host$1";
    }   

    location ~* ^.+\.(ogg|ogv|svg|svgz|eot|otf|woff|mp4|ttf|rss|atom|jpg|jpeg|gif|png|ico|zip|tgz|gz|rar|bz2|doc|xls|exe|ppt|tar|mid|midi|wav|bmp|rtf)$ {
        access_log off; log_not_found off; expires max;
    }

    location = /robots.txt { access_log off; log_not_found off; }
    location ~ /\. { deny  all; access_log off; log_not_found off; }

    # makes for easy testing and logging if cache was hit
    add_header X-Cache $upstream_cache_status;
}
```
