# Hearthstone Deck Helper

Вспомогательный сервис для игроков [Hearthstone](https://playhearthstone.com/) и организаторов фан-турниров.  

> Доступен на http://91.227.18.9/

> Это новая, полностью переписанная версия [старого проекта](https://github.com/ysaron/NeuraHS). Использованы более новые версии Python и Django. Проект докеризован, отрефакторен и избавлен от бесполезных фич.

---

- [Описание](#описание)
  - [Стэк](#стэк)
  - [Функционал](#функционал)
  - [Реализовано](#реализовано)
  - [Иллюстрации](#иллюстрации)
    - [Рендеринг колод (примеры)](#рендеринг-колод)
- [Локальная установка и запуск](#локальная-установка-и-запуск)
  - [Требования](#требования)
  - [Установка](#установка)
  - [Запуск](#запуск)

---

## Описание

**Hearthstone** - коллекционная карточная игра, в которой колоды составляются из 30-40 карт и копируются из клиента игры как байтовые строки в Base64 (чтобы ими было удобно делиться).  
Основная функция сервиса - расшифровка таких строк по известному алгоритму и удобный просмотр колод.  

### Стэк

| **Python 3.10**                 |                                                                   |
| :-----------------------------: | ----------------------------------------------------------------: |
| **Django 4**                    |                                                                   | 
| **Django REST Framework**       | Открытый API (read-only доступ к картам и колодам)                | 
| **PostgreSQL 14**               |                                                                   | 
| **Celery**                      | Рендеринг колод                                                   | 
| **celery-beat**                 | Проверка обновлений Hearthstone; обновление БД                    | 
| **Redis**                       | Бэкенд и брокер для Celery; бэкенд кэширования                    | 
| **Docker** & **docker-compose** | Запуск в контейнере PostgreSQL, Celery, Celery-beat, Redis, Nginx | 
| **Nginx** & **Gunicorn**        | Продакшн                                                          | 
| **HTML** & **CSS**              |                                                                   | 
| **JavaScript**                  | AJAX, анимации колод, удаление пустых полей из GET-запросов и т.д.| 

### Функционал

- **Расшифровка кодов колод и их детализированный просмотр**  
  "Сырой" вид прямо из клиента игры также поддерживается.
- **База данных колод**  
  Расшифрованные колоды сохраняются и доступны для просмотра, в т.ч., посредством API.  
  Отображаются также похожие колоды при наличии таковых в БД.
- **Личное хранилище колод пользователя**[^1]  
  Требует авторизации. Колоды сохраняются в отдельных экземплярах.
- **Подробная информация о составе колоды**  
  Полезно для организаторов турниров с особыми требованиями к составлению колод.
- **[Рендеринг](#рендеринг-колод) подробного изображения колоды в высоком разрешении**  
  Полезно при заявлении колод на турниры.  
  **Процесс получения рендера**:  
  1. Юзер нажимает большую кнопку на странице колоды и задает параметры рендеринга в форме.
  2. Клиент отправляет AJAX-запрос с ID колоды.
  3. Запускается таск Celery, создающий рендер и сохраняющий его в `/media/`.
  4. Клиент получает ответ с `task_id`.
  5. Клиент каждую секунду отправляет AJAX-запрос на URL вида `get_render/<task_id>/`, проверяя статус таска.
  6. Если таск выполнен - его результат используется для вставки рендера на страницу. Среднее время ожидания: 3-5 с.
- **База данных карт Hearthstone**  
  В т.ч. неколлекционные карты, недоступные для включения в колоду.  
  Отображаются также колоды, в которых карта присутствует.  
  БД собирается из открытых API:  
  - https://rapidapi.com/omgvamp/api/hearthstone - данные
  - https://hearthstonejson.com - изображения   
  
  **Процесс обновления БД**:   
  1. Раз в сутки запускается таск Celery-Beat, выполняющий запрос к API с данными и сравнивающий версии.
  2. Если версии различаются - спустя 23 часа запускается таск обновления БД. Задержка обусловлена возможной разницей во времени обновления API данных и API изображений.
  3. Таск обновления БД предварительно также сравнивает версии.  
  
  Возможен запуск вручную: `python manage.py update_db`.  
- **Открытый API для read-only доступа к картам и колодам**
  - Функционал:
    - поиск карт и колод по названию, классам, типам, форматам
    - поиск колод по включенным картам
    - расшифровка кода колоды
  - Документация сгенерирована через Swagger
  - API используется [другим моим проектом](https://github.com/ysaron/hdh-api-bot)
- **Статистика по картам и колодам**

### Реализовано
- **Полная локализация (английский и русский язык)**  
  Переключение языка - на боковой панели.
- **Система аккаунтов** (без использования расширений) 
  - Регистрация с подтверждением email
  - Сброс пароля через email
- **Логирование ошибок в Django Middleware**
- **Кэширование** на уровне представлений (Redis в качестве бэкенда)
- **Юнит-тестирование (pytest)**
- **Celery с Redis в качестве брокера и бэкенда**
  - Рендеринг колоды в таске
  - Периодические проверки обновлений hearthstone API посредством celery-beat
  - Планирование обновления БД
- **Контейнеризация (Docker, docker-compose)**
- **Деплой (Nginx + Gunicorn)**


### Иллюстрации

#### Рендеринг колод

![render01](/pics/render01.png)

---

![render02](/pics/render02.png)

---

![render03](/pics/render03.png)

QR-код содержит код колоды.  

---
  
## Локальная установка и запуск

> (!) Для работы сервису требуется предварительное заполнение БД и скачивание + обработка около 1.5 Гб изображений коллекционных карт, что может занять несколько часов.  
> Изображения скачиваются по просьбе разработчиков [hearthstoneJSON.com](https://hearthstonejson.com/docs/images.html) для уменьшения нагрузки.

### Требования
- Docker
- docker-compose
- Git

### Установка

Клонировать репозиторий:
```shell
git clone https://github.com/ysaron/hearthstone-deck-helper.git
```

В корневом каталоге проекта создать `.env.dev` и задать в нем следующие переменные окружения:
```dotenv
DEBUG=1
SECRET_KEY="<your_secret_key>"
DJANGO_ALLOWED_HOSTS=".localhost 127.0.0.1 [::1]"
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=local_db
SQL_USER=local_user
SQL_PASSWORD=password
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres
X_RAPIDARI_KEY=<your_rapidapi_key>
```

`X_RAPIDARI_KEY` - токен для доступа к API с данными Hearthstone, получить нужно [здесь](https://rapidapi.com/omgvamp/api/hearthstone).

Для работы системы аккаунтов понадобится также электронная почта с настроенным SMTP и доп. переменные окружения:
```dotenv
EMAIL_HOST_USER
EMAIL_HOST_PASSWORD
```

### Запуск

Сборка образа + запуск:
```shell
docker-compose up -d --build
```

Переход в контейнер приложения: 
```shell
docker-compose exec web bash
```

Создание суперпользователя:
```shell
python manage.py createsuperuser
```

Сборка БД с картами + скачивание изображений коллекционных карт:
```shell
python manage.py update_db
```

Добавление в БД "starter pack" с колодами:
```shell
python manage.py import_decks
```

Выход из контейнера
```shell
exit
```

Сайт будет доступен на http://127.0.0.1:8000/.

Остановка:
```shell
docker-compose down
```

[^1]: Поскольку клиент игры на данный момент позволяет сохранять только 27 колод одновременно. Маловато? Маловато.
