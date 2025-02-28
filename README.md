# Документация по развертыванию Django с Nginx и Gunicorn

Это руководство описывает, как развернуть Django-проект с использованием Gunicorn в качестве WSGI-сервера и Nginx в качестве обратного прокси.

---

## Предпосылки

- Рабочий Django-проект.
- Установленные Gunicorn и Nginx.
- Настроенные DNS для домена (например, `yourdomain.com`).

---

## Установка и запуск Gunicorn

Установите Gunicorn:

```bash
pip install gunicorn
```

Примечание:

- project_name — название проекта.
- database_name — имя базы данных.
- user_name — имя пользователя базы данных.
- password — пароль базы данных.
- sammy — имя пользователя Ubuntu (замените на ваше).
- Важно!
- Не рекомендуется выполнять эти действия от имени root. Создайте пользователя с правами sudo (см. инструкцию).

Структура проекта может выглядеть следующим образом:

```bash
├── projectname
│   ├── appname
│   │   ├── __init__.py
│   │   ├── admin.py
│   │   ├── apps.py
│   │   ├── models.py
│   │   ├── views.py
│   │   └── urls.py
│   ├── projectname
│   │   ├── __init__.py
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   ├── templates
│   ├── static
│   ├── media
│   ├── manage.py
│   └── requirements.txt
```

Установка необходимых пакетов
Обновите пакеты и установите необходимые инструменты:

```bash
sudo apt update
sudo apt upgrade
sudo apt install -y python3-pip python-dev-is-python3 libpq-dev vim git htop postgresql postgresql-contrib nginx redis-server
```

Настройка PostgreSQL
1. Подключитесь к PostgreSQL от имени пользователя postgres:

```bash
sudo -u postgres psql
```

2. Создайте базу данных:

```bash
CREATE DATABASE database_name;
```

3. Создайте пользователя базы данных:

```bash
CREATE USER user_name WITH PASSWORD 'password';
```

4. Настройте параметры пользователя:

```bash
ALTER ROLE user_name SET client_encoding TO 'utf8';
ALTER ROLE user_name SET default_transaction_isolation TO 'read committed';
ALTER ROLE user_name SET timezone TO 'UTC';

```

5. Предоставьте пользователю все привилегии на базу данных:

```bash
GRANT ALL PRIVILEGES ON DATABASE database_name TO user_name;
```

6. Выйдите из PostgreSQL:

```bash
\q
```

Клонирование проекта из Git-репозитория
Клонируйте ваш проект:

```bash
git clone your_repository_link
```

Настройка виртуального окружения
1. Перейдите в каталог проекта:

```bash
cd my_project
```

2. Создайте виртуальное окружение:

```bash
python3 -m venv venv
```

3. Активируйте виртуальное окружение и установите зависимости:

```bash
. venv/bin/activate && pip3 install -r requirements.txt
```

Если проект использует дополнительные пакеты, добавьте их в requirements.txt.

Настройка Django-проекта
1. Редактирование settings.py:

- Укажите домен и IP-адреса в списке ALLOWED_HOSTS:

```bash
ALLOWED_HOSTS = ['your_server_domain_or_IP', 'second_domain_or_IP']
```
- Настройте подключение к базе данных:

```bash
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'database_name',
        'USER': 'user_name',
        'PASSWORD': 'password',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}
```
- Задайте путь для статических файлов:

```bash
STATIC_ROOT = BASE_DIR / 'staticfiles'
```

2. Применение миграций:

```bash
python3 manage.py makemigrations
python3 manage.py migrate
```

3. Создание суперпользователя:

```bash
python3 manage.py createsuperuser
```

4. Сбор статических файлов:

```bash
python3 manage.py collectstatic
```

5. Проверка работы сервера:

```bash
python3 manage.py runserver 0.0.0.0:8000
```

Откройте в браузере http://server_ip:8000 для проверки работы.
Важно: В продакшене значение DEBUG должно быть установлено в False, чтобы не раскрывать подробности об ошибках пользователям.

Тестирование Gunicorn
Проверьте запуск проекта через Gunicorn:

```bash
cd ~/myproject
gunicorn --bind 0.0.0.0:8000 myproject.wsgi
```

После проверки остановите Gunicorn (нажмите Ctrl+C) и деактивируйте виртуальное окружение:

```bash
deactivate
```

Настройка Gunicorn с systemd
Для автоматического управления Gunicorn создайте сервисный файл:

1. Откройте файл для редактирования:

```bash
sudo nano /etc/systemd/system/gunicorn.service
```

2. Вставьте следующее содержимое (отредактируйте пути и имя пользователя):

```bash
# Defoult
[Unit]
Description=Ardent ASGI Server
After=network.target

[Service]
User=root
Group=www-data
WorkingDirectory=/root/Api_Ardent
ExecStart=/home/sammy/myproject/venv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/sammy/myproject/myproject.sock myproject.wsgi:application

[Install]
WantedBy=multi-user.target



# Uvicorn
Group=root

Environment="DJANGO_SETTINGS_MODULE=set_app.settings"
ExecStart=/root/Api_Ardent/venv/bin/gunicorn --workers 3 --bind unix:/root/Api_Ardent/set_app/set_app.sock set_app.asgi:application -k uvicorn.workers.UvicornWorker
```

3. Сохраните файл (Ctrl+O, Enter) и выйдите (Ctrl+X).
4. Примените изменения и запустите сервис:

```bash
sudo systemctl daemon-reload
sudo systemctl start gunicorn
sudo systemctl enable gunicorn
```

5. Проверьте статус сервиса:

```bash
sudo systemctl status gunicorn
```

6. Для просмотра логов используйте:

```bash
sudo journalctl -u gunicorn
```

7. Если вносите изменения в файл сервиса, не забудьте выполнить:

```bash
sudo systemctl daemon-reload 
sudo systemctl restart gunicorn
```

При успешном запуске в каталоге проекта появится файл myproject.sock

Настройка Nginx
1. Создайте конфигурационный файл для вашего проекта:

```bash
sudo nano /etc/nginx/sites-available/myproject
```

2. Вставьте следующую конфигурацию (отредактируйте пути и имя домена/IP):

```bash
# Defoult
server {
    server_name server_domain_or_IP;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static/ {
        alias /home/sammy/myproject/staticfiles/;
    }

    location /media/ {
        alias /home/sammy/myproject/media/;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/sammy/myproject/myproject.sock;
    }
}

# How need you
server {
    listen 443 ssl http2;
    server_name pyco.uz www.pyco.uz;

    ssl_certificate /etc/letsencrypt/live/pyco.uz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/pyco.uz/privkey.pem;

    location / {
        include proxy_params;
        proxy_pass http://unix:/root/Api_Ardent/set_app/set_app.sock;
        proxy_read_timeout 60;
        proxy_connect_timeout 60;
        proxy_send_timeout 60;
        client_max_body_size 100M;

        set $cors_origin "";
        if ($http_origin ~* (https://menu-three-green\.vercel\.app|https://waiter-app\.vercel\.app|https://waiter-app-six\.vercel\.app|http://localhost:5173|http://localhost:5174)) {
            set $cors_origin $http_origin;
        }

        add_header 'Access-Control-Allow-Origin' $cors_origin always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, DELETE' always;
        add_header 'Access-Control-Allow-Headers' 'Origin, Content-Type, Authorization, X-CSRFToken' always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;

        if ($request_method = OPTIONS) {
            return 204;
        }
    }

    location /static/ {
        alias /root/Api_Ardent/staticfiles/;
        set $cors_origin "";
        if ($http_origin ~* (https://menu-three-green\.vercel\.app|https://waiter-app\.vercel\.app|https://waiter-app-six\.vercel\.app|http://localhost:5173|http://localhost:5174)) {
            set $cors_origin $http_origin;
        }
        add_header 'Access-Control-Allow-Origin' $cors_origin always;
    }

    location /media/ {
        alias /root/Api_Ardent/media/;
        set $cors_origin "";
        if ($http_origin ~* (https://menu-three-green\.vercel\.app|https://waiter-app\.vercel\.app|https://waiter-app-six\.vercel\.app|http://localhost:5173|http://localhost:5174)) {
            set $cors_origin $http_origin;
        }
        add_header 'Access-Control-Allow-Origin' $cors_origin always;
    }
}


```

3. Активируйте сайт, создав символическую ссылку:

```bash
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled
```

4. Проверьте синтаксис конфигурации Nginx:

```bash
sudo nginx -t
```

5. Перезапустите Nginx:

```bash
sudo systemctl restart nginx
```
6. Дать права доступа

```bash
chmod g+x /home/ubuntu/
chmod g+r /home/ubuntu/
sudo chgrp www-data /home/ubuntu/
```

Настройка HTTPS и исправление CORS в Nginx

Настройка HTTPS с Let's Encrypt

Для обеспечения безопасности соединения используется сертификат Let's Encrypt. В конфигурации Nginx добавлены следующие строки:

```bash
server {
    listen 443 ssl http2;
    server_name pyco.uz www.pyco.uz;

    ssl_certificate /etc/letsencrypt/live/pyco.uz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/pyco.uz/privkey.pem;
```
Эти параметры указывают на путь к SSL-сертификату и закрытому ключу.

Как получить Сертификат HTTPS:
Если сертификат отсутствует, его можно получить с помощью Certbot:

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d pyco.uz -d www.pyco.uz
```

Для автоматического обновления сертификата:

```bash
sudo certbot renew --dry-run
```

Исправление CORS в Nginx

Для разрешения запросов с определённых доменов используется настройка CORS. В конфигурации Nginx реализован динамический механизм установки заголовка Access-Control-Allow-Origin.

Основные CORS-настройки:

```bash
set $cors_origin "";
if ($http_origin ~* (https://menu-three-green\.vercel\.app|https://waiter-app\.vercel\.app|https://waiter-app-six\.vercel\.app|http://localhost:5173|http://localhost:5174)) {
    set $cors_origin $http_origin;
}

add_header 'Access-Control-Allow-Origin' $cors_origin always;
add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, DELETE' always;
add_header 'Access-Control-Allow-Headers' 'Origin, Content-Type, Authorization, X-CSRFToken' always;
add_header 'Access-Control-Allow-Credentials' 'true' always;
```

Этот блок кода:
 1. Проверяет, совпадает ли источник запроса ($http_origin) с одним из разрешённых доменов.
 2. Если да, устанавливает Access-Control-Allow-Origin на $http_origin.
 3. Разрешает использование методов GET, POST, OPTIONS, PUT, DELETE.
 4. Разрешает заголовки Origin, Content-Type, Authorization, X-CSRFToken.
 5. Позволяет передавать куки (Access-Control-Allow-Credentials).

Обработка preflight-запросов (OPTIONS)

```bash
if ($request_method = OPTIONS) {
    return 204;
}
```
Этот блок кода отвечает статусом 204 No Content на preflight-запросы, которые браузер отправляет перед выполнением CORS-запросов.

CORS для статики и медиа

Дополнительно применяются CORS-настройки для статических и медиа-файлов:

```bash
location /static/ {
    alias /root/Api_Ardent/staticfiles/;
    set $cors_origin "";
    if ($http_origin ~* (https://menu-three-green\.vercel\.app|https://waiter-app\.vercel\.app|https://waiter-app-six\.vercel\.app|http://localhost:5173|http://localhost:5174)) {
        set $cors_origin $http_origin;
    }
    add_header 'Access-Control-Allow-Origin' $cors_origin always;
}

location /media/ {
    alias /root/Api_Ardent/media/;
    set $cors_origin "";
    if ($http_origin ~* (https://menu-three-green\.vercel\.app|https://waiter-app\.vercel\.app|https://waiter-app-six\.vercel\.app|http://localhost:5173|http://localhost:5174)) {
        set $cors_origin $http_origin;
    }
    add_header 'Access-Control-Allow-Origin' $cors_origin always;
}
```
Эти настройки позволяют фронтенд-приложениям запрашивать файлы из /static/ и /media/ без проблем с CORS.

# Defoult start

```bash
gunicorn --workers 3 --bind unix:/root/food/set_app/set_app.sock set_app.wsgi:application
```

# Start unicorn
```bash
gunicorn --workers 3 --bind unix:/root/Api_Ardent/set_app/set_app.sock set_app.asgi:application -k uvicorn.workers.UvicornWorker
websocket run
```

# Start stop

```bash
ps aux | grep gunicorn
pkill -9 gunicorn
```

Завершение настройки
Откройте в браузере домен или IP-адрес сервера для проверки развертывания проекта. Рекомендуется также настроить SSL-сертификат (например, с помощью <a href="https://letsencrypt.org/">Let's Encrypt</a>) для обеспечения безопасности соединения.

Дополнительная информация

- <a href="https://docs.djangoproject.com/en/5.1/">Официальная документация Django</a>

Если возникнут вопросы или проблемы, обратитесь к сообществу или на <a href="https://stackoverflow.com/questions">Stack Overflow</a>.

Va nihoyat serverni sozlab boldik!!! Siz brauzeringizda server IP si yo'ki domenini kiritib buni tekshirib ko'rishingiz mumkin, ha aytkancha serveringiz uchun <a href="https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-20-04">SSL sertifikat olish</a>ni unitmang

https://stackoverflow.com/questions/41951792/gunicorn-and-django-error-permission-denied-for-sock

Ba'zi holatlarda 502 xatolikini olishingiz mumkin: https://stackoverflow.com/questions/25774999/nginx-stat-failed-13-permission-denied

NGINX faylni o'zgartirish kerak rootga https://stackoverflow.com/questions/70111791/nginx-13-permission-denied-while-connecting-to-upstream
