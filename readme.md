# Docker project with laravel using php mysql and apche2
##  Project Structure

```
        app
        |__bootstrap
        |__config
        |__database
        |__public
        |__resources
        |__routes
        |__run               (+)
            |__.gitkeep       (+)
        |__storage
        |__tests
        .dockerignore        (+)
        .editorconfig
        .env
        .env.example
        .gitattributes
        .gitignore
        artisan
        CHANGELOG.md
        composer.json
        docker-compose.yml   (+)
        Dockerfile           (+)
        package.json
        phpunit.xml
        readme.md
        server.php
        webpack.mix.js
```

The idea is we'll be builiding the image as well as running docker-compose commands from the main application folder, while run folder contains necessary config and local database for development. With docker volumes, we'll be able to keep the source, the vendor dependencies and local development database in our host, while all the runtime (apache, php) are kept and manged by the container.

## Web server image for laravel 
```
    FROM php:7.2-apache

    RUN apt-get update

    # 1. development packages
    RUN apt-get install -y \
        git \
        zip \
        curl \
        sudo \
        unzip \
        libicu-dev \
        libbz2-dev \
        libpng-dev \
        libjpeg-dev \
        libmcrypt-dev \
        libreadline-dev \
        libfreetype6-dev \
        g++

    # 2. apache configs + document root
    ENV APACHE_DOCUMENT_ROOT=/var/www/html/public
    RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf
    RUN sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf

    # 3. mod_rewrite for URL rewrite and mod_headers for .htaccess extra headers like Access-Control-Allow-Origin-
    RUN a2enmod rewrite headers

    # 4. start with base php config, then add extensions
    RUN mv "$PHP_INI_DIR/php.ini-development" "$PHP_INI_DIR/php.ini"

    RUN docker-php-ext-install \
        bz2 \
        intl \
        iconv \
        bcmath \
        opcache \
        calendar \
        mbstring \
        pdo_mysql \
        zip

    # 5. composer
    COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

    # 6. we need a user with the same UID/GID with host user
    # so when we execute CLI commands, all the host file's ownership remains intact
    # otherwise command from inside container will create root-owned files and directories
    ARG uid
    RUN useradd -G www-data,root -u $uid -d /home/devuser devuser
    RUN mkdir -p /home/devuser/.composer && \
        chown -R devuser:devuser /home/devuser
```
Starting with the webserver itself, php-apache image by default set document root to /var/www/html. However since laravel index.php is inside /var/www/html/public, we need to edit the apache config as well as sites-available. We'll also enable mod_rewrite for url matching and mod_headers for configuring webserver headers.
```
    ENV APACHE_DOCUMENT_ROOT=/var/www/html/public
    RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf
    RUN sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf
```
    Moving onto php configuration, we start by using the provded php.ini, then add a couple of extensions via docker-php-ext-install. The order of doing these tasks are not important (php.ini won't be overwritten) since the configs that loads each extensions are kept in separate files.

    For composer, what we're doing here is fetching the composer binary located at /usr/bin/composer from the composer:latest docker image. Obviously you can specify any other version you want in the tag, instead of latest. This is part of docker's multi-stage build feature.
```
    COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
```
Final steps are optional. Since we're going to mount the application source code from host into the container for development, any command run from within the container CLI shouldn't affect host files/folder ownership. This is helpful for configs and such generated by php artisan. Here I'm using ARG to let other team members set their own uid that matches their host user uid.
```
    ARG uid
    RUN useradd -G www-data,root -u $uid -d /home/devuser devuser
    RUN mkdir -p /home/devuser/.composer && \
        chown -R devuser:devuser /home/devuser
```
## In docker-composer.yml file 
We need to set up composer for connect image and run our application together
### docker-composer.yml file
```
    version: '3.5'

    services:
    laravel-app:
        build:
        context: '.'
        args:
            uid: ${UID}
        container_name: laravel-app
        environment:
        - APACHE_RUN_USER=#${UID}
        - APACHE_RUN_GROUP=#${UID}
        volumes:
        - .:/var/www/html
        ports:
        - 8000:80
        networks:
        backend:
            aliases:
            - laravel-app

    mysql-db:
        image: mysql:5.7
        container_name: mysql-db
        volumes:
        - ./run/var:/var/lib/mysql
        environment:
        - MYSQL_ROOT_PASSWORD=securerootpassword
        - MYSQL_DATABASE=db
        - MYSQL_USER=dbuser
        - MYSQL_PASSWORD=secret
        networks:
        backend:
            aliases:
            - db

    networks:
    backend:
        name: backend-network
```

A few things to go through here. First of all for the laravel container:
* `build:context` refers to the `Dockerfile` that we just written, kept in the same directory as `docker-compose.yml`. 
* `args` is for the `uid` I mentioned above. We'll write `UID` value in the app `.env` file to let docker-compose pick it up.

```
    ...
    MIX_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
    MIX_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"

    UID=1000
```
* `APACHE_RUN_USER` and `APACHE_RUN_GROUP` **ENV** variables comes with php-apache. By doing this, files generated by the webserver will also have consistent ownership.
* `volumes` directive tells docker to mount the host's app source code into `/var/www/html` - which is consistent with **apache** configuration. This enables any change from host files be reflected in the container. Commands such as `composer require` will add **vendor** to host, so we won't need to install dependencies everytime container is brought down and up again.
* If you are building container for CI / remote VM envrionment however, you'll need to add the source files into the container pre-build. For example:
```
    COPY . /var/www/html
    RUN cd /var/www/html && composer install && php artisan key:generate
```
* `ports` is optional, leave out if you're fine with running it under port 80. Alternatively, it can be configurable using `.env` similar to build args:
```
    ports:
    - ${HOST_PORT}:80
```

```
    HOST_PORT=8080
```
* `networks` with `aliases` is also optional. By default, `docker-compose` create a **default** network prefixed with the parent folder name to connect all the services specified in `docker-compose.yml`. However if you have a development of more than 1 `docker-compose`, specifying `networks` name like this allow you to join it from the other `docker-compose.yml` files. `another-app` here will be able to reach **laravel-app** and vice versa, using the specified `aliases`.

`docker-compose.yml`

```
    services:
    another-app:
        networks:
        backend:
            aliases:
            - another-app

    networks:
    backend:
        external:
        name: backend-network
```
#### Now moving onto `mysql`
* `mysql:5.7` is very configurable and just works well out-of-the-box. So we won't need to extend it.
* Simply pick up the `.env` set in laravel app to set username and password for the db user:
```
    environment:
        - MYSQL_ROOT_PASSWORD=securerootpassword
        - MYSQL_DATABASE=${DB_DATABASE}
        - MYSQL_USER=${DB_USERNAME}
        - MYSQL_PASSWORD=${DB_PASSWORD}
```

 
 * Also make sure `.env` `DB_HOST` set to what mysql-db service name, or its aliases: `.env`

 ```
    DB_HOST=mysql-db
 ```

 * Ideally you want to keep database changes in the repository, using a series of migrations and seeders. However if you want to start the mysql container with an existing SQL dump, simply mount the SQL file:

 ```
    volumes:
        - ./run/var:/var/lib/mysql
        - ./run/dump/init.sql:/docker-entrypoint-initdb.d/init.sql
 ```
 * Using `volumes`, we're keeping the database locally under `run/var`, since any data written by `mysqld` is inside the container's `/var/lib/mysql`. We just need to ignore the local database in both `.gitignore` and `.dockerignore` (for build context):
 <br>
 <br>
 
 **.gitignore**

 ```
    /node_modules
    /public/hot
    /public/storage
    /storage/*.key
    /vendor
    .env
    .phpunit.result.cache
    Homestead.json
    Homestead.yaml
    npm-debug.log
    yarn-error.log

    run/var
 ```
 **.dockerignore**
 ```
    run/var
 ```
 ## Up and Running 
 Now let's build the environment, and get it up running. We'll also be installing composer dependencies as well as some artisan command.
