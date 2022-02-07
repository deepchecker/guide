# Процесс обновления DeepChecker

Скачиваем обновленный архив и загружаем его в `/home/deepchecker` с заменой файлов.

```
cd /home/deepchecker
sudo supervisorctl stop all
composer update --ignore-platform-reqs
php artisan migrate
php artisan optimize
sudo supervisorctl start all
```

# Список изменений

## 1.0.1

- Скрыта информация о доступном обновлении для обычных пользователей
