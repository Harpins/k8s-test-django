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


# Развертывание в Minikube

## Предварительные требования

- Установленный [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- Установленный [kubectl](https://kubernetes.io/docs/tasks/tools/)
- Docker Desktop (для Windows) или Docker (для Linux/Mac)
- База данных PostgreSQL, запущенная отдельно (или используйте внешний хост)

# Шаг 1: Запустите Minikube (например, на базе Docker)

```bash
minikube start --driver=docker
```

# Шаг 2: Соберите Docker образ приложения

```bash
docker build -t django_app:latest ./backend_main_django
```

# Шаг 3: Загрузите образ в Minikube

```bash
minikube image load django_app:latest
```

# Шаг 4: Запустите внешнюю БД, например в контейнере Docker

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

# Шаг 5: Настройте конфиги и секреты

Для настройки конфигурации укажите актуальные данные в `k8s/configmap.yaml`. 

Для задания секретов используйте в качестве примера `k8s/secret_example.yaml`. Скопируйте его содержимое в `k8s/secret.yaml` и укажите в новом файле действительные значения переменных `secret-key` и `postgres-password`.

# Шаг 6: Примените манифесты В ПРАВИЛЬНОМ ПОРЯДКЕ

## 1. Создайте пространство имен
```bash
kubectl apply -f k8s/namespace.yaml
```

## 2. Создайте секреты и конфигурацию
```bash
kubectl apply -f k8s/secrets.yaml
kubectl apply -f k8s/configmap.yaml
```

## 3. Разверните приложение
```bash
kubectl apply -f k8s/deployment.yaml
```

## 4. Создайте сервис для доступа извне
```bash
kubectl apply -f k8s/service.yaml
```

## 5. Настройте Ingress (для доступа по домену)
Подробности настройки на Windows [тут](#настройка-ingress-с-туннелированием-на-windows)
```bash
kubectl apply -f k8s/ingress.yaml
```

## 6. Настройте CronJob (автоматическая очистка сессий)
```bash
kubectl apply -f k8s/cronjob.yaml
```
По умолчанию очистка сессий проходит каждый день в 3:00 ночи. Для изменения расписания нужно отредактировать файл `cronjob.yaml`:

```yaml
spec:
  schedule: "0 3 * * *"
```

# Шаг 7: Проверьте статус развертывания и логи пода с приложением

```bash
kubectl get all -n django-app
kubectl logs -n django-app -l app=django
```

# Шаг 8: Выполните миграции в БД и создайте суперпользователя

## Запустите работу, исполняющую миграции и сборку статики

```bash
kubectl apply -f k8s/migrate-and-colstatic-job.yaml
```

## Создайте суперпользователя

```bash
kubectl apply -f k8s\createsuperuser-job.yaml
```

# Настройка Ingress с туннелированием на Windows

### Особенности Minikube на Windows с драйвером Docker

Когда Minikube запущен внутри Docker на Windows, его внутренняя сеть недоступна напрямую из основной операционной системы. Ingress работает, но его IP-адрес виден только внутри контейнера Minikube.

### Решение: Использование Minikube Tunnel

#### 1. Включите Ingress аддон

```bash
minikube addons enable ingress
```

#### 2. Создайте манифест Ingress

В качестве примера используйте `k8s/ingress.yaml`


#### 3. Запустите Minikube Tunnel

Откройте новый терминал PowerShell от имени администратора и выполните:

```bash
minikube tunnel
```

Система может запросить разрешение на доступ к сети - нажмите «Разрешить» или «Да».

Этот терминал должен оставаться открытым всё время работы с приложением!

В этом случае туннелирование будет проводиться на локальный хост `127.0.0.1`

#### 4. Добавьте запись в файл hosts

Откройте файл `C:\Windows\System32\drivers\etc\hosts` из блокнота, запущенного от имени администратора, добавьте в конец строку `127.0.0.1  django.local` и сохраните.

Опционально очистите DNS кэш

```bash
ipconfig /flushdns
```

#### 5. Проверьте доступность

Откройте браузер и перейдите по адресу: `http://django.local`
