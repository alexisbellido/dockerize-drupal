Using Drupal with Docker containers
===========================================================================

A Drupal stack running with Docker.

I will try to have separate containers for MySQL, PHP7-FPM and Nginx.


Overview
===========================================================================

Create a bridge network for all the containers used by this project on your host.

  ``docker network create -d bridge drupal``


Create a MySQL container with one database and database root user.

  ``docker run -d --network=drupal --env MYSQL_ROOT_PASSWORD=root --env MYSQL_DATABASE=drupal --hostname=drupal-db1 --name=drupal-db1 mysql:5.7.17``


Create an Nginx container mapping to a host directory containing the Drupal installation.

  ``mkdir -p /home/alexis/mydocker/drupal-project/html``


This project includes a couple of Nginx configuration files in its nginx directory but you can copy Nginx's default configuration files from a basic Nginx container. First create a basic Nginx container.

  ``docker run -d --name=temp-nginx nginx:1.10.2``
  ``cd /my-nginx-conf``
  ``docker cp temp-nginx:/etc/nginx/conf.d/default.conf /my-nginx-conf/conf.d/``
  ``docker cp temp-nginx:/etc/nginx/nginx.conf /my-nginx-conf/``
  ``docker rm -f temp-nginx``


and then start a container to use those configuration files and easily reconfigure Nginx editing from the host.

  ``docker run -d --network=drupal -v /home/alexis/mydocker/dockerize-drupal/nginx/conf.d/drupal7.conf:/etc/nginx/conf.d/default.conf -v /home/alexis/mydocker/dockerize-drupal/nginx/nginx.conf:/etc/nginx/nginx.conf -v /home/alexis/mydocker/drupal-project/html:/usr/share/nginx/html -p 40010:80 --hostname=drupal-web1 --name=drupal-web1 nginx:1.10.2``


Create container for PHP FPM, which will be called from Nginx.::


    docker run -d --network=drupal -v /home/alexis/mydocker/drupal-project/html:/usr/share/nginx/html --hostname=drupal-php1 --name=drupal-php1 php:7.1.2-fpm

Manually add PHP extensions. This can be incorporated into my PHP Dockerfile later. See https://hub.docker.com/_/php/ for instructions.::

    docker exec -it drupal-php1 bash
    apt-get update
    apt-get install -y libfreetype6-dev libjpeg62-turbo-dev libmcrypt-dev libpng12-dev 
    docker-php-ext-install -j$(nproc) iconv mcrypt
    docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ && docker-php-ext-install -j$(nproc) gd
    docker-php-ext-install -j$(nproc) pdo_mysql


Make sure use the correct Nginx configuration file as some rewrite rules differ for Drupal 7 and 8.

Launch with Docker Compose.::

    cd compose-drupal7
    docker-compose up -d


Troubleshooting
===========================================================================

The Nginx and PHP-FPM containers need to map to the same directory because Nginx only passes a path to PHP-FPM. If you get 404 errors visiting a php file you may need to verify that the SCRIPT_FILENAME parameter is passing the correct path. I temporarily changed the log_format main in nginx.conf to verify the value of $document_root and I noticed it was /etc/nginx/html so I added the root directive to the location block processing php files.::

    location ~ \.php$ {
        try_files $uri =404;
        # root is super important to let the PHP-FPM access the path provided by Nginx
        root /usr/share/nginx/html;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        # this points to the container running PHP-FPM
        fastcgi_pass drupal-php1:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }


Drupal 7 may not detect clean URLs if you installed it without the correct rewrite rules in your Nginx configuration. If that happens. Delete the database and start from scratch making sure the Nginx configuration is correct.
