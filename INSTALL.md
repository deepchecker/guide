# Вводная информация

DeepChecker - приложение, потребляющее большое количество ресурсов. Все операции происходят в десятках отдельных процессов, в связи с чем для корректной работы приложения требуется VPS с не менее чем 4 vCPU и 4Гб ОЗУ.

Приложение во время пиковой нагрузки создает большую нагрузку на сторонние API. В связи с чем сразу после установки, перед добавлением мнемоников рекомендуется арендовать от 10 до 20 http-прокси. Ниже список сервисов, где их можно приобрести:

- https://proxyline.net/ipv4/
- https://proxy6.net/
- https://proxy-seller.ru

# Авторизация

После первой установки будет доступен следующий аккаунт администратора:

Логин: user@site.ru
Пароль: password

# Установка DeepChecker на чистый образ Ubuntu 20.04

## Установка необходимого софта

```
sudo apt install software-properties-common git zip unzip curl
```

## Установка Redis

```
sudo apt install redis-server
```

## Установка MySQL 8

```
sudo apt install mysql-server
sudo mysql_secure_installation
```

Во время выполнения последней команды отвечаем на следующие вопросы:

- Would you like to setup VALIDATE PASSWORD component? **пустая строка**
- New password/Re-enter new password **Новый пароль от БД**
- Remove anonymous users? **Y**
- Disallow root login remotely? **Y**
- Remove test database and access to it? **Y**
- Reload privilege tables now? **Y**

После заходим в БД `mysql -u root -p` и добавляем новую базу данных:

```
mysql> create database deepchecker;
Query OK, 1 row affected (0.00 sec)

mysql> exit;
```

По непонятной причине в дальнейшем (настройка проекта) во время работы с БД может появляться сообщение `SQLSTATE[HY000] [1698] Access denied for user 'root'@'localhost'`. Если столкнулись с такой проблемой, вернитесь к данному шагу и выполните следующие действия.

```
mysql -u root -p
mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'ваш пароль';
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> exit;
```

## Установка PHP 8.0

```
sudo add-apt-repository ppa:ondrej/php
sudo apt update
sudo apt install php8.0-fpm php8.0-mysql php8.0-mbstring php8.0-gmp php8.0-redis php8.0-curl
```

## Установка Nginx

```
sudo apt-get install nginx
cd /etc/nginx/sites-available
sudo rm default
sudo nano default
```

Вставляем данный конфиг, сохраняем, закрываем.

```
server {
    listen 80 default_server;

    return 444;
}

server {
    listen 80;
    server_name ваш домен;
    root /home/deepchecker/public;
    index index.php;

    gzip on;
    gzip_disable "msie6";
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 5;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_min_length 256;
    gzip_types
        application/atom+xml
        application/javascript
        application/json
        application/ld+json
        application/manifest+json
        application/rss+xml
        application/vnd.geo+json
        application/vnd.ms-fontobject
        application/x-font-ttf
        application/x-font-woff
        application/x-web-app-manifest+json
        application/xhtml+xml
        application/xml
        font/opentype
        image/webp
        image/bmp
        image/svg+xml
        image/x-icon
        text/cache-manifest
        text/css
        text/plain
        text/vcard
        text/vnd.rim.location.xloc
        text/vtt
        text/x-component
        text/x-cross-domain-policy;

    add_header "X-Content-Type-Options" "nosniff";
    add_header "X-UA-Compatible" "IE=Edge";
    add_header "X-XSS-Protection" "1; mode=block";
    add_header "Strict-Transport-Security" "max-age=31536000; includeSubDomains";
    add_header "Content-Security-Policy" "default-src * data: blob: 'unsafe-eval' 'unsafe-inline'";
    add_header "Referrer-Policy" "origin";

    client_max_body_size 64m;

    location / {
        try_files $uri $uri/ /index.php?_url=$uri&$query_string;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:/run/php/php8.0-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_index index.php;
        include fastcgi_params;
    }

    location ~ /.well-known {
        allow all;
    }
}
```

## Установка Composer

```
cd /tmp
curl -sS https://getcomposer.org/installer -o composer-setup.php
sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
```

## Установка Supervisor

```
sudo apt install supervisor
```

## Настройка проекта

```
cd /home
mkdir deepchecker
cd deepchecker
```

Скачиваем архив с чекером. Распаковываем его в `/home/deepchecker`

```
composer install --ignore-platform-reqs
cp .env.example .env
php artisan key:generate
```

Открываем файл .env

```
sudo nano /home/deepchecker/.env
```

Заменяем **APP_URL** на адрес сайта, **DB_** на параметры базы данных. После чего выполняем:

```
php artisan config:cache
php artisan migrate
```

## Настройка crontab

```
crontab -e
```

Добавляем:

```
* * * * * cd /home/deepchecker && php artisan schedule:run >> /dev/null 2>&1
```

## Настройка Supervisor

```
sudo nano /etc/supervisor/conf.d/crypt-worker.conf
```

Вставляем

```
[program:crypt-worker-1]
process_name=%(program_name)s_%(process_num)02d
command=php /home/deepchecker/artisan wallet:parse 1 --loop
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=1
redirect_stderr=true
stdout_logfile=/home/deepchecker/storage/logs/worker.log
stopwaitsecs=3600

[program:crypt-worker-2]
process_name=%(program_name)s_%(process_num)02d
command=php /home/deepchecker/artisan wallet:parse 2 --loop
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=1
redirect_stderr=true
stdout_logfile=/home/deepchecker/storage/logs/worker.log
stopwaitsecs=3600

[program:crypt-worker-3]
process_name=%(program_name)s_%(process_num)02d
command=php /home/deepchecker/artisan wallet:parse 3 --loop
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=1
redirect_stderr=true
stdout_logfile=/home/deepchecker/storage/logs/worker.log
stopwaitsecs=3600

[program:crypt-worker-4]
process_name=%(program_name)s_%(process_num)02d
command=php /home/deepchecker/artisan wallet:parse 4 --loop
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=1
redirect_stderr=true
stdout_logfile=/home/deepchecker/storage/logs/worker.log
stopwaitsecs=3600

[program:crypt-worker-5]
process_name=%(program_name)s_%(process_num)02d
command=php /home/deepchecker/artisan wallet:parse 5 --loop
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=1
redirect_stderr=true
stdout_logfile=/home/deepchecker/storage/logs/worker.log
stopwaitsecs=3600

[program:crypt-worker-6]
process_name=%(program_name)s_%(process_num)02d
command=php /home/deepchecker/artisan wallet:parse 6 --loop
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=1
redirect_stderr=true
stdout_logfile=/home/deepchecker/storage/logs/worker.log
stopwaitsecs=3600

[program:crypt-worker-7]
process_name=%(program_name)s_%(process_num)02d
command=php /home/deepchecker/artisan wallet:parse 7 --loop
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=1
redirect_stderr=true
stdout_logfile=/home/deepchecker/storage/logs/worker.log
stopwaitsecs=3600

[program:crypt-worker-8]
process_name=%(program_name)s_%(process_num)02d
command=php /home/deepchecker/artisan wallet:parse 8 --loop
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=1
redirect_stderr=true
stdout_logfile=/home/deepchecker/storage/logs/worker.log
stopwaitsecs=3600

[program:crypt-worker-9]
process_name=%(program_name)s_%(process_num)02d
command=php /home/deepchecker/artisan wallet:parse 9 --loop
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=1
redirect_stderr=true
stdout_logfile=/home/deepchecker/storage/logs/worker.log
stopwaitsecs=3600

[program:crypt-worker-10]
process_name=%(program_name)s_%(process_num)02d
command=php /home/deepchecker/artisan wallet:parse 10 --loop
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=1
redirect_stderr=true
stdout_logfile=/home/deepchecker/storage/logs/worker.log
stopwaitsecs=3600

[program:crypt-addresses-worker-1]
process_name=%(program_name)s_%(process_num)02d
command=php /home/deepchecker/artisan wallet:addresses 1 2 --loop
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=1
redirect_stderr=true
stdout_logfile=/home/deepchecker/storage/logs/worker.log
stopwaitsecs=3600

[program:crypt-addresses-worker-2]
process_name=%(program_name)s_%(process_num)02d
command=php /home/deepchecker/artisan wallet:addresses 3 4 --loop
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=1
redirect_stderr=true
stdout_logfile=/home/deepchecker/storage/logs/worker.log
stopwaitsecs=3600

[program:crypt-addresses-worker-3]
process_name=%(program_name)s_%(process_num)02d
command=php /home/deepchecker/artisan wallet:addresses 5 6 --loop
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=1
redirect_stderr=true
stdout_logfile=/home/deepchecker/storage/logs/worker.log
stopwaitsecs=3600

[program:crypt-addresses-worker-4]
process_name=%(program_name)s_%(process_num)02d
command=php /home/deepchecker/artisan wallet:addresses 7 8 --loop
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=1
redirect_stderr=true
stdout_logfile=/home/deepchecker/storage/logs/worker.log
stopwaitsecs=3600

[program:crypt-addresses-worker-5]
process_name=%(program_name)s_%(process_num)02d
command=php /home/deepchecker/artisan wallet:addresses 9 10 --loop
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
numprocs=1
redirect_stderr=true
stdout_logfile=/home/deepchecker/storage/logs/worker.log
stopwaitsecs=3600
```

Выполняем

```
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start all
```

## Установка сертификата

```
sudo apt insatll snapd
sudo snap install core
sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx
```

```
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): **Ваш email**
(Y)es/(N)o: **Y**
(Y)es/(N)o: **Y**
Select the appropriate numbers separated by commas and/or spaces, or leave input
blank to select all options shown (Enter 'c' to cancel): **Номер слева от вашего домена**
```
