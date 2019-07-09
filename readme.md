# Docker project with laravel using php mysql and apche2
## Project Structure

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