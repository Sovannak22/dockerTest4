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