https://www.webgis.dev/posts/laravel-php-and-docker

<p align="center"><a href="https://laravel.com" target="_blank"><img src="https://raw.githubusercontent.com/laravel/art/master/logo-lockup/5%20SVG/2%20CMYK/1%20Full%20Color/laravel-logolockup-cmyk-red.svg" width="400" alt="Laravel Logo"></a></p>

<p align="center">
<a href="https://github.com/laravel/framework/actions"><img src="https://github.com/laravel/framework/workflows/tests/badge.svg" alt="Build Status"></a>
<a href="https://packagist.org/packages/laravel/framework"><img src="https://img.shields.io/packagist/dt/laravel/framework" alt="Total Downloads"></a>
<a href="https://packagist.org/packages/laravel/framework"><img src="https://img.shields.io/packagist/v/laravel/framework" alt="Latest Stable Version"></a>
<a href="https://packagist.org/packages/laravel/framework"><img src="https://img.shields.io/packagist/l/laravel/framework" alt="License"></a>
</p>

## About Laravel

Laravel is a web application framework with expressive, elegant syntax. We believe development must be an enjoyable and creative experience to be truly fulfilling. Laravel takes the pain out of development by easing common tasks used in many web projects, such as:

- [Simple, fast routing engine](https://laravel.com/docs/routing).
- [Powerful dependency injection container](https://laravel.com/docs/container).
- Multiple back-ends for [session](https://laravel.com/docs/session) and [cache](https://laravel.com/docs/cache) storage.
- Expressive, intuitive [database ORM](https://laravel.com/docs/eloquent).
- Database agnostic [schema migrations](https://laravel.com/docs/migrations).
- [Robust background job processing](https://laravel.com/docs/queues).
- [Real-time event broadcasting](https://laravel.com/docs/broadcasting).

Laravel is accessible, powerful, and provides tools required for large, robust applications.

## Learning Laravel

Laravel has the most extensive and thorough [documentation](https://laravel.com/docs) and video tutorial library of all modern web application frameworks, making it a breeze to get started with the framework.

You may also try the [Laravel Bootcamp](https://bootcamp.laravel.com), where you will be guided through building a modern Laravel application from scratch.

If you don't feel like reading, [Laracasts](https://laracasts.com) can help. Laracasts contains over 2000 video tutorials on a range of topics including Laravel, modern PHP, unit testing, and JavaScript. Boost your skills by digging into our comprehensive video library.

## Laravel Sponsors

We would like to extend our thanks to the following sponsors for funding Laravel development. If you are interested in becoming a sponsor, please visit the [Laravel Partners program](https://partners.laravel.com).

### Premium Partners

- **[Vehikl](https://vehikl.com/)**
- **[Tighten Co.](https://tighten.co)**
- **[WebReinvent](https://webreinvent.com/)**
- **[Kirschbaum Development Group](https://kirschbaumdevelopment.com)**
- **[64 Robots](https://64robots.com)**
- **[Curotec](https://www.curotec.com/services/technologies/laravel/)**
- **[Cyber-Duck](https://cyber-duck.co.uk)**
- **[DevSquad](https://devsquad.com/hire-laravel-developers)**
- **[Jump24](https://jump24.co.uk)**
- **[Redberry](https://redberry.international/laravel/)**
- **[Active Logic](https://activelogic.com)**
- **[byte5](https://byte5.de)**
- **[OP.GG](https://op.gg)**

## Contributing

Thank you for considering contributing to the Laravel framework! The contribution guide can be found in the [Laravel documentation](https://laravel.com/docs/contributions).

## Code of Conduct

In order to ensure that the Laravel community is welcoming to all, please review and abide by the [Code of Conduct](https://laravel.com/docs/contributions#code-of-conduct).

## Security Vulnerabilities

If you discover a security vulnerability within Laravel, please send an e-mail to Taylor Otwell via [taylor@laravel.com](mailto:taylor@laravel.com). All security vulnerabilities will be promptly addressed.

## License

The Laravel framework is open-sourced software licensed under the [MIT license](https://opensource.org/licenses/MIT).

## DEVELOPMENT 
You will first need to install docker and docker-compose on your development machine. I strongly recommend you use a Linux machine; the code below is all using Ubuntu 22.04

First, let's create a new Laravel application using composer. If composer is installed on your development host, you can use it directly. Otherwise, you can use the compose Docker official image to do it:

This command runs the official composer image in interactive mode (--interactive) 
in a pseudo-TTY (--tty) 
with the current directory mapped to /app in the container (--volume $PWD:/app) 
with the current user context (--user $(id -u):$(id -g)).
It runs the composer command: "composer create-project laravel/laravel laravel-gis"
which will install Laravel with composer in a directory called larave-gis
The --rm flag will remove the container once the command as finished

docker run --rm --interactive --tty \
	--volume $PWD:/app \
	--user $(id -u):$(id -g) \
	composer create-project laravel/laravel laravel-gis
Create a few extra needed files, folders, and permissions in the new project directory:

cd laravel-gis
touch docker-compose.yml
mkdir -p docker/nginx
touch docker/nginx/nginx-site.conf
sudo chown -R 33:33 . (you can do this better inside the container's console)
sudo chmod -R 777 .
We use some pretty dangerous permission settings here. Still, it's a development environment, and we will need write/execute permissions on pretty much all of our project files and directories. We will see how to set up security permissions for our production environment in a future post. Don't use this setup in production

We will need two docker services to get our environment working with php-fpm and nginx; create two services in the docker-compose.yml file:

version: "3.7"
networks:
    frontend:
    backend:
services:
### nginx proxy service (based on nginx official image) to act as a web server and proxy server
    proxy:
        image: nginx:latest
### map local port 8080 to containers's port 80
		ports:
            - "8080:80"
### map the current directory to /var/www/app in the container
### and map our the nginx-site.conf to the nginx default site in the container
        volumes:
            - ./:/var/www/app
            - ./docker/nginx/nginx-site.conf:/etc/nginx/conf.d/default.conf
        networks:
            - frontend
            - backend
### php-fpm service (based on php official image) to process our php code 
	php:
        image: php:8.1-fpm
### map the current directory to /var/www/app in the container (the same as for the proxy service)
        volumes:
            - ./:/var/www/app
        networks:
            - backend
Put the following content in the docker/nginx/nginx-site.conf file:

server {
    listen 80;
### our root directory points to the public Laravel directory
    root /var/www/app/public;
    index index.php;
    server_name _;
		
### Redirects all queries to routes without extension (/dashboard for instance) to their
### equivalents but with a trailing /index.php (/dashboard/index.php) so it hits the next location
    location / {
         try_files $uri $uri/ /index.php$is_args$args;
    }
		
### Sends all php queries to the php container (named php in our case) on port 9000
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
Run "docker-compose up" and go to http://localhost:8080 in your browser; you should see the blank laravel welcome page confirming that everything is working properly:
https://github.com/raycastsant/assets/blob/main/a659f575-5958-4536-9534-13d258dfdedd.png

(windows tip for windows port issues)
net stop winnat
docker start container_name
net start winnat
