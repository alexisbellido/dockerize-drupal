Using Drupal with Docker containers
===========================================================================

A Drupal stack running with Docker.

I will try to have separate containers for MySQL, PHP7-FPM and Nginx.


Overview
===========================================================================

Create a bridge network for your containers on your host.

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

  ``docker run -d --network=drupal -v /home/alexis/mydocker/dockerize-drupal/nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf -v /home/alexis/mydocker/dockerize-drupal/nginx/nginx.conf:/etc/nginx/nginx.conf -v /home/alexis/mydocker/drupal-project/html:/usr/share/nginx/html -p 40010:80 --hostname=drupal-web1 --name=drupal-web1 nginx:1.10.2``
