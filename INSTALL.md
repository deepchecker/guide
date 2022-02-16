# Вводная информация

DeepChecker - приложение, потребляющее большое количество ресурсов. Все операции происходят в десятках отдельных процессов, в связи с чем для корректной работы приложения требуется VPS с не менее чем 4 vCPU и 4Гб ОЗУ. ОС: Ubuntu 20.04
Приобрести VPS можно здесь:
- https://www.cherryservers.com/
- https://www.digitalocean.com/
- https://www.ovhcloud.com/
- https://www.hetzner.com/


Приложение во время пиковой нагрузки создает большую нагрузку на сторонние API. В связи с чем сразу после установки, перед добавлением Seed-фраз рекомендуется добавить через админ-панель чекера от 10 до 20 ipv4 http-прокси (желательно приватных). Ниже список сервисов, где их можно приобрести:

- https://proxyline.net/ipv4/
- https://proxy6.net/
- https://proxy-seller.ru

# Доменное имя

Для полноценной работы DeepChecker - необходимо доменное имя с установленным SSL сертификатом. Софт способен работать и на ip адресе, но в таком случае - вы будете лишены возможности получать уведомления от телеграм бота, поскольку API telegram работает только по протоколу https.

Купите домен у любого удобного вам регистратора и добавьте A-запись с IP вашего VPS сервера.

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

- Would you like to setup VALIDATE PASSWORD component? **N**
- New password/Re-enter new password **Вводим желаемый пароль от БД**
- Remove anonymous users? **Y**
- Disallow root login remotely? **Y**
- Remove test database and access to it? **Y**
- Reload privilege tables now? **Y**

Не забудьте сохранить пароль от БД, он еще пригодится.  
После заходим в БД `mysql -u root -p` и добавляем новую базу данных:

```
mysql> create database deepchecker;
Query OK, 1 row affected (0.00 sec)

mysql> exit;
```

В дальнейшем (настройка проекта) во время работы с БД может появляться сообщение `SQLSTATE[HY000] [1698] Access denied for user 'root'@'localhost'`. Если столкнулись с такой проблемой, вернитесь к данному шагу и выполните следующие действия.

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

Вставляем данный конфиг, заменяем 'yourdomain.com' на свой домен

```
server {
    listen 80 default_server;

    return 444;
}

server {
    listen 80;
    server_name yourdomain.com;
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
Жмем Ctrl + x, затем Y, затем Enter

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
sudo mkdir deepchecker
cd deepchecker
```

Загружаем архив с чекером на сервер в папку `/home/deepchecker`.  

Распаковываем архив так, чтобы внутри папки deepchecker оказались папки app, bootstrap, config, database, public и другие файлы архива.
```
unzip deepchecker.zip
```

Права доступа к файлам

```
chmod -R 775 /home/deepchecker
sudo chgrp -R www-data /home/deepchecker/storage /home/deepchecker/bootstrap/cache
sudo chmod -R ug+rwx /home/deepchecker/storage /home/deepchecker/bootstrap/cache
```

```
composer install --ignore-platform-reqs
cp .env.example .env
php artisan key:generate
```

Открываем файл .env

```
sudo nano /home/deepchecker/.env
```

Заменяем значение **APP_URL** на ваш домен формате https://domain.com/ (обязателньо через https)  
Заменяем значение **DB_PASSWORD** на пароль, который вы указывали выше (на шаге 'Установка MySQL 8')  


После чего выполняем:

```
composer clearcache
composer update --ignore-platform-reqs
php artisan config:cache
php artisan migrate
```

## Настройка crontab

```
crontab -e
```
Choose 1-4 [1]: **1**  

Добавляем в конец файла:

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
sudo apt install snapd
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

# Авторизация

После первой установки вам доступен следующий аккаунт администратора:

Логин: user@site.ru
Пароль: password

# Начало работы

## Прокси
Не забудьте добавить ранее купленные прокси через админ - панель, перейдя по ссылке /proxies  

## Телеграм - бот
- Пишем в личные сообщения тг **@botfather**, создаем бота, получаем токен бота в формате 111111111:DASKDI2109290DWR10-4013389120-32  
- Заходим в админку чекера по ссылке **/options**, заполняем поля 'Имя бота (указываем без @)' и 'Токен'.  
- Пишем **/start** в личные сообщения нашему боту.  
- Возвращаемся в админку чекера на страницу **/settings**. Заполняем поле 'Уведомлять, если баланс кошелька превысит' пороговой суммой в $.  
- Копируем текст из поля **/pair 0000000000000000000000000** и отправляем его боту.  
- Если все верно - бот пришлет вам сообщение 'Привязка успешно совершена' и с этого момента будет присылать уведомления о находках.

# Можете приступать к работе.
