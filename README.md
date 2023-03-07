# проект Yamdb API

![CI/CD](https://github.com/mikepavlos/yamdb_final/actions/workflows/yamdb_workflow.yml/badge.svg)

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

Проект разворачивается в контейнерах Docker, с использованием технологий CI/CD 
(continuous integration и continuous deployment).  
При внесении изменений в код проекта и отправке в репозиторий GitHub, производится автоматическая проверка на соответствие стандарту PEP8, тестирование Pytest.  
После успешного прохождения тестов обновленный Docker-образ сохраняется на DockerHub и разворачивается в Docker-контейнерах на сервере.

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

### Запуск проекта локально в Docker-контейнерах:

Перейти в директорию `infra/`, создать файл `.env`, заполнить его по образцу:

```text
DB_ENGINE=django.db.backends.postgresql # указываем, что работаем с postgresql
DB_NAME=postgres                        # имя базы данных
POSTGRES_USER=postgres                  # логин для подключения к базе данных
POSTGRES_PASSWORD=postgres              # пароль для подключения к БД (установите свой)
DB_HOST=db                              # название сервиса (контейнера)
DB_PORT=5432                            # порт для подключения к БД
```

Выполнить команду развертывания контейнеров проекта:

```commandline
docker-compose up -d
```

### Запуск проекта на сервере:

```editorconfig
infra/
  nginx/
    default.conf
  .env
  docker-compose.yaml
```

- `.env` (заполнить по образцу выше)
- `docker-compose.yaml`
```yaml
version: '3.8'

volumes:
  db_data:
  static_value:
  media_value:

services:

  db:
    image: postgres:13.0-alpine
    volumes:
      - db_data:/var/lib/postgresql/data/
    env_file:
      - .env

  web:
    image: miha1is/api_yamdb:v1
    restart: always
    volumes:
      - static_value:/app/static/
      - media_value:/app/media/
    depends_on:
      - db
    env_file:
      - .env

  nginx:
    image: nginx:1.21.3-alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
      - static_value:/var/html/static/
      - media_value:/var/html/media/
    depends_on:
      - web
```
- папка с файлом `/nginx/default.conf`

```editorconfig
server {
    server_tokens off;

    listen 80;

    server_name 127.0.0.1;

    location /static/ {
        root /var/html/;
    }

    location /media/ {
        root /var/html/;
    }

    location / {
        proxy_pass http://web:8000;
    }
}
```

В директории `infra/` выполнить команду развертывания контейнеров проекта:

```commandline
docker-compose up -d
```

---

После успешного развертывания пректа, выполнить миграции, 
создать суперпользователя, собрать статику:

```commandline
docker-compose exec web python manage.py migrate
```

```commandline
docker-compose exec web python manage.py createsuperuser
```

```commandline
docker-compose exec web python manage.py collectstatic --no-input 
```

Для сохранения данных базы: 

```commandline
docker-compose exec web python manage.py dumpdata > fixtures.json
```

и последующего ее заполнения:

```commandline
docker cp fixtures.json <container_id>:/app
```

```commandline
docker-compose exec web python manage.py shell  
>>> from django.contrib.contenttypes.models import ContentType
>>> ContentType.objects.all().delete()
>>> quit()

docker-compose exec web python manage.py loaddata fixtures.json
```

---

### Примеры работы с API Yatube:

#### Основные endpoints проекта:

**Категории**  
/api/v1/categories/

**Жанры**  
/api/v1/genres/

**Произведения**  
/api/v1/titles/

**Пользователи**  
/api/v1/users/

**Рецензии**  
/api/v1/titles/(?P<title_id>\d+)/reviews/

**Комментарии к рецензиям**  
/api/v1/titles/(?P<title_id>\d+)/reviews/(?P<review_id>\d+)/comments/

**Аутентификация**  
/api/v1/auth/signup/

**Получение токена по почте для аутентификации**  
/api/v1/auth/token/

#### Алгоритм регистрации пользователей

- Пользователь отправляет POST-запрос на добавление нового пользователя с параметрами `email` и `username` на эндпоинт /api/v1/auth/signup/.
- YaMDB отправляет письмо с кодом подтверждения (`confirmation_code`) на адрес email.
- Пользователь отправляет POST-запрос с параметрами `username` и `confirmation_code` на эндпоинт /api/v1/auth/token/, в ответе на запрос ему приходит `token` (JWT-токен).
- При желании пользователь отправляет PATCH-запрос на эндпоинт /api/v1/users/me/ и заполняет поля в своём профайле (описание полей — в документации).

#### Пользовательские роли

- Аноним — может просматривать описания произведений, читать отзывы и комментарии.
- Аутентифицированный пользователь (`user`) — может, как и Аноним, читать всё, дополнительно он может публиковать отзывы и ставить оценку произведениям (фильмам/книгам/песенкам), может комментировать чужие отзывы; может редактировать и удалять свои отзывы и комментарии. Эта роль присваивается по умолчанию каждому новому пользователю.
- Модератор (`moderator`) — те же права, что и у Аутентифицированного пользователя плюс право удалять любые отзывы и комментарии.
- Администратор (`admin`) — полные права на управление всем контентом проекта. Может создавать и удалять произведения, категории и жанры. Может назначать роли пользователям.
- Суперюзер Django — обладет правами администратора (`admin`)

#### Некоторые примеры запросов к API и ответов от сервера:

##### Пример 1 (Получение публикаций)

**Регистрация нового пользователя**  
```
POST http://127.0.0.1:8000/api/v1/auth/signup/
```
*Тело запроса:*  
```
{
  "email": "user@example.com",
  "username": "string"
}
```

**Ответ сервера**  
```
{
  "email": "string",
  "username": "string"
}
```
##### Пример 2 (Получение списка всех категорий)

**Запрос к API**  
```
GET http://127.0.0.1:8000/api/v1/categories/
```
**Ответ сервера**  
```
{
  "count": 0,
  "next": "string",
  "previous": "string",
  "results": [
    {
      "name": "string",
      "slug": "string"
    }
  ]
}
```
##### Пример 3 (Добавление новой категории)

**Запрос к API**  
```
POST http://127.0.0.1:8000/api/v1/categories/
```
*Тело запроса:*  
```
{
  "name": "string",
  "slug": "string"
}
```
**Ответ сервера**  
```
{
  "name": "string",
  "slug": "string"
}
```
##### Пример 4 (Получение списка всех жанров)

**Запрос к API**  
```
GET http://127.0.0.1:8000/api/v1/genres/
```
**Ответ сервера**  
```
{
  "count": 0,
  "next": "string",
  "previous": "string",
  "results": [
    {
      "name": "string",
      "slug": "string"
    }
  ]
}
```
##### Пример 5 (Добавление жанра)

**Запрос к API**  
```
POST http://127.0.0.1:8000/api/v1/genres/
```
*Тело запроса:*  
```
{
  "name": "string",
  "slug": "string"
}
```
**Ответ сервера**  
```
{
  "name": "string",
  "slug": "string"
}
```
##### Пример 6 (Получение списка всех произведений)

**Запрос к API**  
```
GET http://127.0.0.1:8000/api/v1/titles/
```
**Ответ сервера**  
```
{
  "count": 0,
  "next": "string",
  "previous": "string",
  "results": [
    {
      "id": 0,
      "name": "string",
      "year": 0,
      "rating": 0,
      "description": "string",
      "genre": [
        {
          "name": "string",
          "slug": "string"
        }
      ],
      "category": {
        "name": "string",
        "slug": "string"
      }
    }
  ]
}
```
##### Пример 7 (Получение информации о произведении)

**Запрос к API**  
```
GET http://127.0.0.1:8000/api/v1/titles/{titles_id}/
```
**Ответ сервера**  
```
{
  "id": 0,
  "name": "string",
  "year": 0,
  "rating": 0,
  "description": "string",
  "genre": [
    {
      "name": "string",
      "slug": "string"
    }
  ],
  "category": {
    "name": "string",
    "slug": "string"
  }
}
```
##### Пример 8 (Добавление произведения)

**Запрос к API**  
```
POST http://127.0.0.1:8000/api/v1/titles/
```
*Тело запроса:*  
```
{
  "name": "string",
  "year": 0,
  "description": "string",
  "genre": [
    "string"
  ],
  "category": "string"
}
```
**Ответ сервера**  
```
{
  "id": 0,
  "name": "string",
  "year": 0,
  "rating": 0,
  "description": "string",
  "genre": [
    {
      "name": "string",
      "slug": "string"
    }
  ],
  "category": {
    "name": "string",
    "slug": "string"
  }
}
```
##### Пример 9 (Получение списка всех отзывов)

**Запрос к API**  
```
GET http://127.0.0.1:8000/api/v1/titles/{title_id}/reviews/
```
**Ответ сервера**  
```
{
  "count": 0,
  "next": "string",
  "previous": "string",
  "results": [
    {
      "id": 0,
      "text": "string",
      "author": "string",
      "score": 1,
      "pub_date": "2019-08-24T14:15:22Z"
    }
  ]
}
```
##### Пример 10 (Частичное обновление отзыва по id)

**Запрос к API**  
```
POST http://127.0.0.1:8000/api/v1/titles/{title_id}/reviews/{review_id}/
```
*Тело запроса:*  
```
{
  "text": "string",
  "score": 1
}
```
**Ответ сервера**  
```
{
  "id": 0,
  "text": "string",
  "author": "string",
  "score": 1,
  "pub_date": "2019-08-24T14:15:22Z"
}
```

##### Пример 11 (Добавление комментария к отзыву)

**Запрос к API**  
```
POST http://127.0.0.1:8000/api/v1/titles/{title_id}/reviews/{review_id}/comments/
```
*Тело запроса:*  
```
{
  "text": "string"
}
```
**Ответ сервера**  
```
{
  "id": 0,
  "text": "string",
  "author": "string",
  "pub_date": "2019-08-24T14:15:22Z"
}
```
#### Более подробные примеры запросов и ответов к endpoints проекта описаны в документации проекта, доступной по адресу:
(после запуска проекта)
http://localhost/redoc/


### Авторы проекта:

- Тимлид-разработчик: Михаил Павлов https://github.com/mikepavlos

- Разработчик: Даниил Чебенев https://github.com/ZonRedress

- Разработчик: Марк Ройтблат (ex-Фролов) https://github.com/frolovmark

