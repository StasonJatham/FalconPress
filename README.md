# FalconPress

Deploy WordPress with the highest performance options currently possible, in Docker-Compose.

## TODO
 - [ ] add reverse proxy
 - [ ] do something with the logs (ingest them somewhere)
 - [ ] add hardware/server reource monitoring
 - [ ] set up caching plugins on install
 - [ ] add watchtowerrr for automatic updating
 - [ ] better log structure

 ## Bugs
 - logging not working correctly so far


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
    volumes:
      - ${PERSISTENT_DATA}/${WORDPRESS_DB_HOST}/:/var/lib/mysql
      - ${LOG_PATH}/${WORDPRESS_DB_HOST}/general-log.log:/var/lib/mysql/general-log.log
    environment:
      - MARIADB_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
    command: mysqld --general-log=1 --general-log-file=/var/lib/mysql/general-log.log

  falcon-cache:
    image: redis:latest
    restart: unless-stopped
    depends_on:
      - "falcon-db"
    volumes:
      - ${PERSISTENT_DATA}/${REDIS_HOST}:/data
    command: redis-server --requirepass $${REDIS_HOST_PASSWORD}

  falconpress:
    image: "wordpress:${WORDPRESS_VERSION}"
    volumes:
      - ${STATIC_ROOT}:/var/www/html
      - ${CONFIG_DIR}/falconpress/z-update-php-mem-limit.ini:/usr/local/etc/php/conf.d/z-update-php-memory-limit.ini
    depends_on:
      - "falcon-db"
      - "falcon-cache"
    environment:
      - WORDPRESS_DB_HOST=${WORDPRESS_DB_HOST}
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - WORDPRESS_DB_NAME=${MYSQL_DATABASE}
      - WORDPRESS_DB_USER=${MYSQL_USER}
      - WORDPRESS_DB_PASSWORD=${MYSQL_PASSWORD}
      - WORDPRESS_TABLE_PREFIX=${WORDPRESS_TABLE_PREFIX}
      - |
        WORDPRESS_CONFIG_EXTRA=
        define( 'WP_REDIS_HOST', '$${REDIS_HOST}' );
        define( 'WP_REDIS_PORT', 6379 );
        define( 'WP_REDIS_PASSWORD', '$${REDIS_HOST_PASSWORD}' );
    restart: unless-stopped

  falcon-web:
    image: nginx:latest
    depends_on:
      - "falconpress"
      - "falcon-db"
      - "falcon-cache"
    volumes:
      - ${CONFIG_DIR}/nginx:/etc/nginx/conf.d
      - ${LOG_PATH}/nginx:/var/log/nginx
      - ${STATIC_ROOT}:/var/www/html
    ports:
      - 8080:80
```

# Future
- [ ] Add custom FalconCache Plugin (handles CDN, Cloudflare, Nginx everythng)
- [ ] Add custom FalconTheme
- [ ] Add static site generation (Nextjs/Gatsby or maybe standard?)

