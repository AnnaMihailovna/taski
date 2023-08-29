##  Tasking
### Описание
Приложение для планирования задач. Есть фронтенд (SPA-приложение на React)
и бэкенд (Django-приложение). Задачи можно добавлять, изменять, удалять
и переводить из группы "незавершённые" в "завершённые".

### Технологический стек
* Python 3.9
* Django REST
* Gunicorn
* Nginx
* JS
* Node.js

### Деплой
Подключитесь к удалённому серверу
```
ssh -i путь_до_SSH_ключа/название_файла_с_SSH_ключом_без_расширения login@ip
```
Клонируйте проект на сервер
```
git clone git@github.com:AnnaMihailovna/tasking-django-react.git
```
Переходим в директорию backend-приложения проекта
```
cd tasking-django-react/backend/
```
Создаём виртуальное окружение
```
python -m venv venv
```
Активируем виртуальное окружение
```
source venv/bin/activate
```
Обновляем pip в виртуальном окружении
```
pip install --upgrade pip
```
Устанавливаем зависимости
```
pip install -r requirements.txt
```
Из директории, в которой находится файл manage.py применяем миграции
```
python manage.py migrate
```
Создаём суперпользователя
```
python manage.py createsuperuser
```
Собираем статику бэкенда
```
python manage.py collectstatic
```
Из корня проекта скопируем статику бэкенда в системную директорию
```
sudo cp -r /home/yc-user/tasking-django-react/backend/static_backend/ /var/www/taski/
```
Запускаем веб-сервер разработки Django
```
python manage.py runserver
```
В файле settings.py xxx.xxx.xxx.xxx укажите IP вашего сервера
```
ALLOWED_HOSTS = ['xxx.xxx.xxx.xxx', '127.0.0.1', 'localhost']
```
В другом окне терминала установите зависимости для фронтенд-приложения. Перейдите в директорию tasking-django-react/frontend/ и выполните команду
```
npm i
```
Запустите приложение командой
```
npm run start
```
Проверте тестовый запуск в браузере по адресу
http://внешний_ip_адрес_сервера:3000

### Установка и запуск Gunicorn
На удалённом сервере при активированном виртуальном окружении проекта
```
pip install gunicorn==20.1.0
```
Из директории с файлом manage.py
```
gunicorn --bind 0.0.0.0:8000 backend.wsgi
```
Проверим на админке - должна работать без статики
```
http://ваш_публичный_IP:8000/admin/
```
Остановим и запустим для непрерывной работы.

В директории /etc/systemd/system/ создайте файл gunicorn.service и откройте его в Nano
```
sudo nano /etc/systemd/system/gunicorn.service
```
Подставьте в код из листинга свои данные, добавьте этот код без комментариев в файл gunicorn.service и сохраните изменения
```
[Unit]
# Это текстовое описание юнита, пояснение для разработчика.
Description=gunicorn daemon 

# Условие: при старте операционной системы запускать процесс только после того, 
# как операционная система загрузится и настроит подключение к сети.
# Ссылка на документацию с возможными вариантами значений 
# https://systemd.io/NETWORK_ONLINE/
After=network.target 

[Service]
# От чьего имени будет происходить запуск:
# укажите имя, под которым вы подключались к серверу.
User=yc-user 

# Путь к директории проекта:
# /home/<имя-пользователя-в-системе>/
# <директория-с-проектом>/<директория-с-файлом-manage.py>/.
# Например:
WorkingDirectory=/home/yc-user/tasking-django-react/backend/

# Команду, которую вы запускали руками, теперь будет запускать systemd:
# /home/<имя-пользователя-в-системе>/
# <директория-с-проектом>/<путь-до-gunicorn-в-виртуальном-окружении> --bind 0.0.0.0:8000 backend.wsgi
ExecStart=/home/yc-user/tasking-django-react/backend/venv/bin/gunicorn --bind 0.0.0.0:8000 backend.wsgi

[Install]
# В этом параметре указывается вариант запуска процесса.
# Значение <multi-user.target> указывают, чтобы systemd запустил процесс,
# доступный всем пользователям и без графического интерфейса.
WantedBy=multi-user.target
```
Чтобы точно узнать путь до Gunicorn, активируйте виртуальное окружение и воспользуйтесь командой
```
which gunicorn
```
Заново запустите процесс gunicorn.service
```
sudo systemctl start gunicorn 
```
Добавьте процесс Gunicorn в список автозапуска операционной системы на удалённом сервере
```
sudo systemctl enable gunicorn 
```
### Установка Nginx
Находясь на удалённом сервере, из любой директории
```
sudo apt install nginx -y
sudo systemctl start nginx
```
Укажите файрволу, какие порты должны остаться открытыми
```
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH
```
Включите файрвол
```
sudo ufw enable
```
Проверьте внесённые изменения
```
sudo ufw status
```
Запустите сборку фронтенд-приложения из директории tasking-django-react/frontend/
```
npm run build
```
Скопируйте в системную директорию Nginx (которую он использует по умолчанию для доступа к статическим файлам — /var/www/) содержимое папки .../frontend/build/
```
sudo cp -r /home/yc-user/tasking-django-react/frontend/build/. /var/www/taski/
```
Через редактор Nano откройте файл конфигурации веб-сервера
```
sudo nano /etc/nginx/sites-enabled/default
```
Удалите все настройки из файла, запишите и сохраните новые
```
server {

    listen 80;
    server_name публичный_ip_вашего_удалённого_сервера;
    
    location /api/ {
        # Эта команда определяет, куда нужно перенаправить запрос.
        proxy_pass http://127.0.0.1:8000;
    }

    location /admin/ {
        proxy_pass http://127.0.0.1:8000;
    }

    location / {
        root   /var/www/taski;
        index  index.html index.htm;
        try_files $uri /index.html;
    }

}
```
Проверьте файл конфигурации на ошибки
```
sudo nginx -t
```
Перезагрузите конфигурацию Nginx
```
sudo systemctl reload nginx
```
В адресную строку браузера введите внешний IP вашего удалённого сервера без указания порта.

Команда для просмотра лога последних запросов
```
sudo tail /var/log/nginx/access.log
```
### Над проектом работала(бэкенд и деплой)
[AnnaMihailovna](https://github.com/AnnaMihailovna/)
