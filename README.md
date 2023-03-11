# проект Yamdb API ![CI/CD](https://github.com/mikepavlos/yamdb_final/actions/workflows/yamdb_workflow.yml/badge.svg)

### Описание проекта:

Проект YaMDb собирает отзывы пользователей на произведения.  
Сами произведения в YaMDb не хранятся, здесь нельзя посмотреть фильм или послушать музыку.  
Произведения делятся на категории, такие как «Книги», «Фильмы», «Музыка». 
Список произведений, категорий и жанров может быть расширен администратором.  

Восхищенные или возмущённые пользователи оставляют к произведениям текстовые отзывы и ставят произведению оценку в диапазоне от одного до десяти (целое число); 
из пользовательских оценок формируется усреднённая оценка произведения — рейтинг (целое число). 
На одно произведение пользователь может оставить только один отзыв.
Пользователи могут оставлять комментарии к отзывам.  
Добавлять отзывы, комментарии и ставить оценки могут только аутентифицированные пользователи.  

Сервис развернут на сервере, доступен по адресу: http://158.160.59.172/api/v1  
Документация: http://158.160.59.172/redoc/

---

Проект разворачивается в контейнерах Docker из образов приложения 'web', официальных 'db' (postgres) и 'nginx', с использованием технологий CI/CD 
(continuous integration и continuous deployment) GitHub Actions.  
При внесении изменений в код проекта и отправке в репозиторий GitHub, производится автоматическая проверка на соответствие стандарту PEP8, тестирование Pytest.  
После успешного прохождения тестов обновленный Docker-образ 'web' сохраняется на DockerHub и разворачивается в Docker-контейнерах на сервере.

### Стек технологий:

- Python 3.7
- Django 2.2
- Django REST framework
- Django rest_framework_simplejwt
- Gunicorn
- Nginx
- PostgreSQL
- Git
- Docker

---

### Получение проекта:

Клонировать репозиторий:

```commandline
https://github.com/mikepavlos/yamdb_final.git
```
---

### Запуск проекта локально в Docker-контейнерах:

Перейти в директорию `infra/`, создать файл `.env`, заполнить его по образцу:

```text
DB_ENGINE=django.db.backends.postgresql # указываем, что работаем с postgresql
DB_NAME=postgres                        # имя базы данных
POSTGRES_USER=postgres                  # логин для подключения к базе данных
POSTGRES_PASSWORD=postgres              # пароль для подключения к БД (установите свой)
DB_HOST=db                              # название сервиса (контейнера)
DB_PORT=5432                            # порт для подключения к БД
SECRET_KEY=                             # секретный ключ джанго-проекта
ALLOWED_HOSTS=localhost                 # локальный хост, 127.0.0.1, [::1] и т.п.
```

Выполнить команду развертывания контейнеров проекта:

```commandline
docker-compose up -d
```

---

### Запуск проекта на сервере:

#### Подготовка GitHub Actions:

Во вкладке настроек репозитория проекта, в меню выбрать `Secrets and variables` выбрать `Actions`, нажав кнопку `New repository secret`, создать переменные окружения:

```
SECRET_KEY             # "секретный ключ джанго-проекта" в кавычках
ALLOWED_HOSTS          # *

DOCKER_USERNAME        # имя пользователя в DockerHub
DOCKER_PASSWORD        # пароль доступа в DockerHub

DB_ENGINE              # django.db.backends.postgresql
DB_NAME                # имя базы данных
POSTGRES_USER          # логин для подключения к базе данных
POSTGRES_PASSWORD      # пароль для подключения к БД
DB_HOST                # db
DB_PORT                # порт для подключения к БД 5432

USER                   # логин сервера
HOST                   # ip сервера
SSH_KEY                # приватный ключ локальной машины, 
                       # по которому происходит вход на сервер (~/.ssh/id_rsa)
PASSPHRASE             # фраза-пароль ключа ssh (если установлена)

TELEGRAM_TO            # id телеграм-чата (@userinfobot)
TELEGRAM_TOKEN         # токен телеграм-бота (@BotFather - /mybots - Choose a bot - API Token)
```

#### Подготовка сервера:

Войти на свой удаленный сервер.
Установить Docker и docker-compose.

```commandline
sudo apt install docker.io
```

```commandline
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

```commandline
sudo chmod +x /usr/local/bin/docker-compose
```

Из паки `infra` локального проекта cкопировать на сервер в домашнюю дирректорию файлы настроек nginx и docker-compose

```commandline
scp docker-compose.yaml <USER>@<HOST>:/home/<USER>
```

```commandline
scp ./nginx/default.conf <USER>@<HOST>:/home/<USER>/nginx/
```

Запуск проекта на сервере командой

```commandline
sudo docker-compose up -d
```

Либо при `push` на GitHub проект тестируется, обновляется его образ и автоматически разворачивается на сервере, о чем приходит сообщение на telegram.

--- 
После успешного развертывания проекта на сервере или локально:  
Выполнить миграции, создать суперпользователя, собрать статику

```commandline
sudo docker-compose exec web python manage.py migrate
```

```commandline
sudo docker-compose exec web python manage.py createsuperuser
```

```commandline
sudo docker-compose exec web python manage.py collectstatic --no-input 
```

Для сохранения данных базы: 

```commandline
sudo docker-compose exec web python manage.py dumpdata > fixtures.json
```

и последующего ее восстановления/заполнения:

```commandline
# скопировать дамп в контейнер 'web'
docker cp fixtures.json <container_id>:/app
```

```commandline
sudo docker-compose exec web python manage.py shell  
>>> from django.contrib.contenttypes.models import ContentType
>>> ContentType.objects.all().delete()
>>> quit()

# выполнить восстановление из дампа
sudo docker-compose exec web python manage.py loaddata fixtures.json
```

---

### Работа с сервисом:

Сервис будет доступен:  

при локальном развертывании - http://localhost/api/v1  
Документация - http://localhost/redoc  

При развертывании на сервере - http://<ip_адрес_хоста>/api/v1  
Документация - http://<ip_адрес_хоста>/redoc

---

### Авторы проекта:

- Тимлид-разработчик, контейнеризация, DevOps сервиса: Михаил Павлов https://github.com/mikepavlos

- Разработчик: Даниил Чебенев https://github.com/ZonRedress

- Разработчик: Марк Ройтблат (ex-Фролов) https://github.com/frolovmark

