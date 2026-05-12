# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

В случае возникновения проблем из-за различия в форматах окончаний строк между Windows и Linux (\r) при запуске ./manage.py, можно преобразовать файл прямо в контейнере

```shell
docker compose run --rm web bash -c "sed -i 's/\r$//' manage.py && ./manage.py migrate"
docker compose run --rm web bash -c "sed -i 's/\r$//' manage.py && ./manage.py createsuperuser"
```


Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8080/admin/](http://127.0.0.1:8080/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


## Развертывание в Minikube

### Предварительные требования

- Установленный [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- Установленный [kubectl](https://kubernetes.io/docs/tasks/tools/)
- Docker Desktop (для Windows) или Docker (для Linux/Mac)
- База данных PostgreSQL, запущенная отдельно (или используйте внешний хост)

### Шаг 1: Запустите Minikube (например, на базе Docker)

```bash
minikube start --driver=docker
```

### Шаг 2: Соберите Docker образ приложения

```bash
docker build -t django_app:latest ./backend_main_django
```

### Шаг 3: Загрузите образ в Minikube

```bash
minikube image load django_app:latest
```

### Шаг 4: Запустите внешнюю БД, например в контейнере Docker

```bash
docker run -d \
  --name postgres-external \
  -p 5432:5432 \
  -e POSTGRES_DB=django_db \
  -e POSTGRES_USER=django_user \
  -e POSTGRES_PASSWORD=postgres123 \
  postgres:12.0-alpine
```
Для доступа из Minikube используйте внутренний хост докера `host.docker.internal`

### Шаг 5: Настройте конфиги и секреты

Для настройки конфигурации укажите актуальные данные в `k8s/configmap.yaml`. 

Для задания секретов используйте в качестве примера `k8s/secret_example.yaml`. Скопируйте его содержимое в `k8s/secret.yaml` и укажите в новом файле действительные значения переменных `secret-key` и `postgres-password`.

### Шаг 6: Примените манифесты В ПРАВИЛЬНОМ ПОРЯДКЕ

# 1. Создайте пространство имен
kubectl apply -f k8s/namespace.yaml

# 2. Создайте секреты и конфигурацию
kubectl apply -f k8s/secrets.yaml
kubectl apply -f k8s/configmap.yaml

# 3. Разверните приложение
kubectl apply -f k8s/deployment.yaml

# 4. Создайте сервис для доступа извне
kubectl apply -f k8s/service.yaml

### Шаг 7: Проверьте статус развертывания и логи пода с приложением

```bash
kubectl get all -n django-app
kubectl logs -n django-app -l app=django
```

### Шаг 8: Выполните миграции в БД

# Найдите имя пода с джанго-приложением и запомните. В последующих командах замените `<pod-name>` на актуальное

```bash
kubectl get pods -n django-app
```

# Выполните миграции

```bash
kubectl exec -n django-app <pod-name> -- python manage.py migrate
```

# Создайте суперпользователя

```bash
kubectl exec -n django-app -it <pod-name> -- python manage.py createsuperuser
```

# Соберите статические файлы (опционально)

```bash
kubectl exec -n django-app <pod-name> -- python manage.py collectstatic --noinput
```

### Шаг 9: Получите доступ к приложению

Доступ к приложению можно получить,например, через minikube-туннель. 

```bash
minikube service django-service -n django-app
```