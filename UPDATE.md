### Процесс обновления DeepChecker

Скачиваем обновленный архив и загружаем его в `/home/deepchecker` с заменой файлов.

```
cd /home/deepchecker
sudo supervisorctl stop all
sudo find . -type f -exec chmod 644 {} \;
sudo find . -type d -exec chmod 755 {} \;
sudo chmod -R 777 ./storage
sudo chmod -R 777 ./bootstrap/cache/
composer clearcache
composer update --ignore-platform-reqs
php artisan migrate
php artisan view:cache
php artisan optimize
sudo supervisorctl start all
```

### Список изменений

### 1.1.0

- Добавлено отображение прогресса добавления seed-фраз
- Возможность скопировать seed-фразу в буфер обмена по клику на фразу
- Возможность привязать к профилю несколько telegram-аккаунтов
- Корректировка логики расчета времени до окончания всех операций
- Добавлен экспорт seed-фраз/адресов
- Настройки приложения теперь объединены с настройками пользователя (но по-прежнему работают глобально и редактируются только администратором)
- Увеличен лимит на максимально возможное количество добавляемых за раз seed-фраз
- Доработки UI

**Внимание!** После обновления (не чистой установки) до данной версии сделайте следующее:

``` 
sudo nano /etc/php/8.0/fpm/php.ini
```

- Значение строки `max_execution_time` заменяем на `1200`
- Значение строк `upload_max_filesize` и `post_max_size` заменяем на `64M`

``` 
sudo nano /etc/nginx/sites-available/default
```

Находим

``` 
location ~ \.php$ {
    try_files $uri =404;
    fastcgi_pass unix:/run/php/php8.0-fpm.sock;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_index index.php;
    include fastcgi_params;
}
```

Заменяем на 

``` 
location ~ \.php$ {
    try_files $uri =404;
    fastcgi_pass unix:/run/php/php8.0-fpm.sock;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_index index.php;
    include fastcgi_params;
    fastcgi_read_timeout 1200;
}
```

```
service php8.0-fpm restart
service nginx restart
```

#### 1.0.23

- Настройки доступных путей для каждой валюты
- Скрыта общая очередь у обычных пользователей
- Предварительная скорость окончания проверки основывается на данных одной очереди (раньше - на среднем времени всех очередей)
- Добавлена возможность загружать seed-фразы в формате TXT
- Добавлена множественная загрузка прокси
- Настройки Telegram-бота теперь необязательны
- Другие изменения и багфиксы

#### 1.0.1

- Скрыта информация о доступном обновлении для обычных пользователей
