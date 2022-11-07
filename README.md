# FalconPress

Deploy WordPress with the highest performance options currently possible, in Docker-Compose.

## MariaDB
Better performance than MySQL.

## Nginx 
Better performance than Apache. Also using the FastCGI Cache for static file cache of wordpress.

## Wordpress
The main thingy. Using the Alpine PHP-FPM Version.

## Redis
Using Redis as Object-Cache between Wordpress and Database
(under construction)

## Docker Compose File
```yaml
version: "3.8"
services:
  falcon-db:
    image: mariadb:latest
    restart: unless-stopped
    env_file:
      - ../configs/.db.env
      - ../configs/.env
    volumes:
      - "${PERSISTENT_DATA}/mariadb/:/var/lib/mysq"
      - "${LOG_PATH}/mariadb/:/var/log/mysql"
      - ../configs/mariadb/:/etc/mysql
    environment:
      - MARIADB_ROOT_PASSWORD="${MYSQL_ROOT_PASSWORD}"
      - MYSQL_DATABASE="${MYSQL_DATABASE}"
      - MYSQL_USER="${MYSQL_USER}"
      - MYSQL_PASSWORD="${MYSQL_PASSWORD}"
    command: mysqld --general-log=1 --general-log-file=/var/lib/mysql/general-log.log

  falcon-cache:
    image: redis:latest
    restart: unless-stopped
    depends_on:
      - "falcon-db"
    env_file:
      - ../configs/.cache.env
      - ../configs/.env
    volumes:
      - ${PERSISTENT_DATA}/redis:/data
    command: redis-server --requirepass $${REDIS_HOST_PASSWORD}

  falconpress:
    image: wordpress:php8.1-fpm-alpine
    volumes:
      - ${STATIC_ROOT}:/var/www/html
      - ../configs/z-update-php-mem-limit.ini:/usr/local/etc/php/conf.d/z-update-php-memory-limit.ini
    depends_on:
      - "falcon-db"
      - "falcon-cache"
    env_file:
      - ../configs/.db.env
      - ../configs/.env
    environment:
      - WORDPRESS_DB_HOST=falcon-db
      - MYSQL_ROOT_PASSWORD="${MYSQL_ROOT_PASSWORD}"
      - WORDPRESS_DB_NAME="${MYSQL_DATABASE}"
      - WORDPRESS_DB_USER="${MYSQL_USER}"
      - WORDPRESS_DB_PASSWORD="${MYSQL_PASSWORD}"
      - WORDPRESS_TABLE_PREFIX=kack_
    restart: unless-stopped

  falcon-web:
    image: nginx:latest
    depends_on:
      - "falconpress"
      - "falcon-db"
      - "falcon-cache"
    env_file:
      - ../configs/.env
    volumes:
      - ../configs/nginx:/etc/nginx/conf.d
      - ${LOG_PATH}/nginx:/var/log/nginx
      - ${STATIC_ROOT}:/var/www/html
    ports:
      - 8080:80

```