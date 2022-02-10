For lazy people here is the docker image link: https://hub.docker.com/repository/docker/harshalone/laravel-9-prod

If you want to understand then keep reading


# Laravel9 : build production docker image
How to build laravel docker image for production use?
I have searched a lot on internet about how to build a real production docker image for Laravel 9 project but so far I have not seen proper solution therefore I am writing this readme to help out those seeking to build real production docker image for their laravel 9 project.

When building a production docker image for laravel project it is very important to create non-root user to run nginx. Let's see how we can  do that create your laravel 9 project first.

# Install Laravel 9 in a folder of your choice
Run following commands

```
composer create-project laravel/laravel:^8.0 example-app
 
cd example-app
 
php artisan serve

```

# How to create Dockerfile for Laravel 9 production?

First of all lets decide on what we need in our production to run our laravel app. We need following things:

- nginx
- php8-fpm
- supervisor



# What is supervisor?

Supervisor is a process manager it keeps background processes up and running. Why we need it? When we build our docker image we will run multiple processes and we want to keep these processes up all the time.

Laravel supervisor will make sure that these processes are up and running and monitor them. We can access these process logs as well as in our docker image.

Let's look at the following diagram to see what processes we would be running via supervisor.


Let's start adding files so that you will know what we are doing.

Create laravel 9 dockerfile

First of all, let's create a dockerfile for our production use. Go to your laravel 9 project root folder and create a file called Dockerfile with following contents:

```
FROM php:8.0-fpm

# Set working directory
WORKDIR /var/www

# Add docker php ext repo
ADD https://github.com/mlocati/docker-php-extension-installer/releases/latest/download/install-php-extensions /usr/local/bin/

# Install php extensions
RUN chmod +x /usr/local/bin/install-php-extensions && sync && \
    install-php-extensions mbstring pdo_mysql zip exif pcntl gd memcached

# Install dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    libpng-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    locales \
    zip \
    jpegoptim optipng pngquant gifsicle \
    unzip \
    git \
    curl \
    lua-zlib-dev \
    libmemcached-dev \
    nginx

# Install supervisor
RUN apt-get install -y supervisor

# Install composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Clear cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Add user for laravel application
RUN groupadd -g 1000 www
RUN useradd -u 1000 -ms /bin/bash -g www www

# Copy code to /var/www
COPY --chown=www:www-data . /var/www

# add root to www group
RUN chmod -R ug+w /var/www/storage

# Copy nginx/php/supervisor configs
RUN cp docker/supervisor.conf /etc/supervisord.conf
RUN cp docker/php.ini /usr/local/etc/php/conf.d/app.ini
RUN cp docker/nginx.conf /etc/nginx/sites-enabled/default

# PHP Error Log Files
RUN mkdir /var/log/php
RUN touch /var/log/php/errors.log && chmod 777 /var/log/php/errors.log

# Deployment steps
RUN composer install --optimize-autoloader --no-dev
RUN chmod +x /var/www/docker/run.sh

EXPOSE 80
ENTRYPOINT ["/var/www/docker/run.sh"]

```


What are we doing in this file:

- we are using php8-fpm official docker image
- defining /var/www as our project root directory
- installing necessary php extentions
- installing supervisor and composer
- creating a www user to run our nginx as www user
- copying all laravel project files to /var/www directory with proper permissions
- applying correct write permissions for our storage folder
- copying configurations file for
1. supervisor
2. php
3. nginx

finally running composer to install required project dependencies
setting entrypoint for our docker project
Add configuration files for php, nginx and supervisor
Our Dockerfile requires some of the configuration files. 

Let's create a directory called docker inside your laravel 9 project and create following files:

```
# Directory structure
|- docker
    |- nginx.conf
    |- php.ini
    |- supervisor.conf
    |- run.sh
We need all files mentioned above to complete our docker image build. Let's add contents for each of the above file:

docker/nginx.conf:
server {
    listen 80;
    root /var/www/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php index.html;
    charset utf-8;

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_buffering off;
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
        gzip_static on;
	}
    
    location ~ /\.(?!well-known).* {
        deny all;
    }
}
docker/php.ini:
log_errors=1
display_errors=1
post_max_size=40M
upload_max_filesize=40M
display_startup_errors=1
error_log=/var/log/php/errors.log
docker/supervisor.conf:
[supervisord]
nodaemon=true
loglevel = info
logfile=/var/log/supervisord.log
pidfile=/var/run/supervisord.pid

[group:laravel-worker]
priority=999
programs=nginx,php8-fpm,laravel-schedule,laravel-notification,laravel-queue

[program:nginx]
priority=10
autostart=true
autorestart=true
stderr_logfile_maxbytes=0
stdout_logfile_maxbytes=0
stdout_events_enabled=true
stderr_events_enabled=true
command=/usr/sbin/nginx -g 'daemon off;'
stderr_logfile=/var/log/nginx/error.log
stdout_logfile=/var/log/nginx/access.log

[program:php8-fpm]
priority=5
autostart=true
autorestart=true
stderr_logfile_maxbytes=0
stdout_logfile_maxbytes=0
command=/usr/local/sbin/php-fpm -R
stderr_logfile=/var/log/nginx/php-error.log
stdout_logfile=/var/log/nginx/php-access.log

[program:laravel-schedule]
numprocs=1
autostart=true
autorestart=true
redirect_stderr=true
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/artisan schedule:run
stdout_logfile=/var/log/nginx/schedule.log

[program:laravel-notification]
numprocs=1
autostart=true
autorestart=true
redirect_stderr=true
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/artisan notification:worker
stdout_logfile=/var/log/nginx/notification.log

[program:laravel-queue]
numprocs=5
autostart=true
autorestart=true
redirect_stderr=true
process_name=%(program_name)s_%(process_num)02d
stdout_logfile=/var/log/nginx/worker.log
command=php /var/www/artisan queue:work sqs --sleep=3 --tries=3
In above file we are doing following things:

creating different programs
each program can run one or more processes
each program have their own log files that we can use to check process logs
we are running following processes in background
laravel schedular
laravel notifications
laravel queues
docker/run.sh:
#!/bin/sh

cd /var/www

# php artisan migrate:fresh --seed
php artisan cache:clear
php artisan route:cache

/usr/bin/supervisord -c /etc/supervisord.conf
run.sh is basically our entrypoint for the container. We are performing some of the necessary deployment steps here for laravel project:
```

running migrations
clearing route and other cache
finally running supervisor deamon
Build your laravel 8 docker image for production
Finally, once you have added all above files you are ready to build your own laravel 8 production docker image. You can build your new docker image using following command.

# Build your docker image
# syntax: docker build -t <image-tag> <dockerfile-location>
docker build -t app:latest .

# If you want to test your local image
docker run -d -p 80:80 app:latest

# once you run above command go to http://localhost
In this tutorial, we covered on how to safely build laravel 8 docker image as non-root user with proper project permissions. You will still need to learn on how to deploy this image to production environment.

I will write another tutorial on how we can deploy this image on aws or gcp environment. You can access above files in my gitlab repo:
