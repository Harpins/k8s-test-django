# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри репозитория две директории: 

`dev` - сайт подготовлен к развертыванию в кластер Yandex Cloud. [Работающая версия сайта](https://edu-igor-derevnin.yc-sirius-dev.pelid.team)

`local` - сайт подготовлен для локального развертывания в minikube.


# Локальная разработка (вне контейнеров, `local\backend_main_django`)

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

Чтобы обновить приложение до последней версии подтяните код из центрального окружения, выберите необходимую версию (local - для локальной разработки и тестировании в minikube, dev - для деплоя в yandex cloud) перейдите в соответствующую папку командой `cd local` или `cd dev` пересоберите докер-образы:

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
## Переменные окружения для локального тестирования приложения

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

# Шаг 4: Запустите и настройте Postgres

## а. Во внешнем контейнере docker:

```bash
docker run -d \
  --name postgres-external \
  -p 5432:5432 \
  -e POSTGRES_DB=django_db \
  -e POSTGRES_USER=django_user \
  -e POSTGRES_PASSWORD=postgres123 \
  postgres:12.0-alpine
```
Для доступа из Minikube используйте в конфиге внутренний хост докера `host.docker.internal`

## б. Развертывание PostgreSQL в кластере с помощью Helm

Helm — это пакетный менеджер для Kubernetes, который позволяет устанавливать сложные приложения (в том числе, PostgreSQL) одной командой, вместо создания множества YAML-файлов вручную.

### Установка Helm на Windows с помощью Chocolatey

В Windows PowerShell (WIN+X) выполните строго от имени администратора:
```bash
choco install kubernetes-helm -y
```

Bitnami больше не поддерживает классический Helm-репозиторий, поэтому устанавливаем Postgres однострочной командой WPS (НЕ от администратора):

```bash
helm install postgres oci://registry-1.docker.io/bitnamicharts/postgresql --namespace django-app --create-namespace --set auth.postgresPassword=yourstrongpassword --set auth.database=django_db --set auth.username=django_user --set primary.persistence.size=10Gi
```

Проверка установки:

```bash
kubectl get pods -n django-app 
kubectl get svc -n django-app
```

В списках подов и сервисов должны появиться связанные с postgres-postgresql.

#### ВАЖНО - Получите реальный пароль от БД и обновите секреты.

Когда вы устанавливаете PostgreSQL через Helm, даже если вы указали свой пароль, Helm может сгенерировать случайный пароль для пользователя БД. Для получения реального пароля выполните в WPS построчно:

```
$encoded = kubectl get secret --namespace django-app postgres-postgresql -o jsonpath="{.data.password}"
$bytes = [System.Convert]::FromBase64String($encoded)
$password = [System.Text.Encoding]::UTF8.GetString($bytes)
Write-Host "Пароль: $password"
```

Реальный пароль необходимо подставить в `secret.stringData.postgres-password`, после чего обновить манифест с секретами.

#### Укажите хост БД configmap.yaml - POSTGRES_HOST:postgres-postgresql.django-app.svc.cluster.local


# Шаг 5: Настройте конфиги и секреты

Для настройки конфигурации укажите актуальные данные в `minikube/configmap.yaml`. 

Для задания секретов используйте в качестве примера `minikube/secret_example.yaml`. Скопируйте его содержимое в `minikube/secret.yaml` и укажите в новом файле действительные значения переменных `secret-key` и `postgres-password`.

# Шаг 6: Примените манифесты В ПРАВИЛЬНОМ ПОРЯДКЕ

## 1. Определите пространство имен
```bash
kubectl apply -f minikube/namespace.yaml
```

## 2. Примените секреты и конфигурацию
```bash
kubectl apply -f minikube/secrets.yaml
kubectl apply -f minikube/configmap.yaml
```

## 3. Разверните приложение
```bash
kubectl apply -f minikube/deployment.yaml
```

## 4. Создайте сервис для доступа извне
```bash
kubectl apply -f minikube/service.yaml
```

## 5. Настройте Ingress (для доступа по домену)
Подробности настройки на Windows [тут](#настройка-ingress-с-туннелированием-на-windows)
```bash
kubectl apply -f minikube/ingress.yaml
```

## 6. Настройте CronJob (автоматическая очистка сессий)
```bash
kubectl apply -f minikube/cronjob.yaml
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
kubectl apply -f minikube/migrate-and-colstatic-job.yaml
```

## Создайте суперпользователя

```bash
kubectl apply -f minikube\createsuperuser-job.yaml
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

В качестве примера используйте `minikube/ingress.yaml`


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

Откройте браузер и перейдите по адресу: http://django.local

# Деплой в кластер Yandex Cloud

## Предварительные требования

### Ресурсы Yandex Cloud

#### 1. В рамках работы с Yandex Cloud в вашем [личном пространстве имен](https://yandex.cloud/ru/docs/managed-kubernetes/operations/kubernetes-cluster/kubernetes-cluster-namespace-create) должны быть выделены и предварительно настроены следующие ресурсы:

* [Домен в связке с Yandex Application Load Balancer (ALB)](https://yandex.cloud/ru/docs/tutorials/web/application-load-balancer-website/console) 

* [Managed PostgreSQL](https://yandex.cloud/ru/docs/managed-postgresql/)
Для подключения к БД в секретах вашего пространства имен должны содержаться данные следующего характера:

- host	
- port	
- name	
- username	
- password	
- root.crt	
- dsn	
- driver
- parameters

Посмотреть секреты Postgres:

```kubectl get secret postgres -n your-namespace -o yaml```

* [Yandex Object Storage (S3)](https://yandex.cloud/ru/docs/storage/concepts/object) - для хранения статики и медиафайлов 

Данные для доступа хранятся в секрете bucket.

- access_key	
- secret_key	
- bucket_name	
- endpoint_url	
- bucket_url	
- dsn	

* [Предустановленный Nginx с Ingress](https://purpleschool.ru/knowledge-base/kubernetes/work-with-components/kubernetes-ingress-nginx) - дефолтная версия. Для деплоя требуется настраивать самостоятельно.


#### 2. Установка Yandex Cloud CLI (YC CLI)

Первый шаг — установить на локальную машину утилиту `yc`, через которую будет проводиться управление ресурсами.
[Руководство](https://yandex.cloud/ru/docs/cli/quickstart)

После установки необходимо инициализировать CLI, авторизоваться, создать профиль, выбрать облако, кластер и настроить зону доступности. Процесс настройки запускается одной командой в интерактивном режиме:
```yc init```

Для отдельного проекта рекомендуется создать новый профиль:
```yc config profile create my-profile```


#### 3. Подключение к готовому кластеру.

Убедитесь, что активен правильный профиль: Выполните ```yc config list``` и проверьте, что выбранное облако и кластер внутри него соответствуют вашему проекту.

Загрузите настройки для доступа к кластеру:
```yc managed-kubernetes cluster get-credentials --id cluster_id --external```

В Windows конфигурации хранятся по умолчанию по пути `C:\Users\User\.kube\config`

Проверьте подключение к кластеру:
```kubectl cluster-info```

Если команда выполнилась успешно и вы увидели информацию о мастере, то все настроено верно.

#### 4. Публикация Docker образа

Перед публикацией необходимо собрать Docker образ Django приложения. Перейдите в директорию \dev\backend_main_django` и выполните команду:

```docker build -t your_docker/django_app:your_tag -f backend_main_django/Dockerfile backend_main_django/```

Авторизуйтесь в докере и опубликуйте образ.

```bash
docker login
docker push your_docker/django_app:your_tag
```
Для отслеживания версий рекомендуется использовать хэш Git коммита.


#### 5. Манифесты и порядок деплоя

Все манифесты и настройки находятся в директории `k8s_yc_deploy`

**Перед деплоем обязательно проверьте и обновите во всех манифестах кластерозависимые параметры: namespace, image, ALLOWED_HOSTS и др. переменные окружения.**

После используйте команду  
```
kubectl apply -f <имя_файла.yaml> -n your-namespace
```
для применения манифестов в следующем порядке:

1.	configmap.yaml	
2.  secret.yaml	- создайте и заполните на примере secret_example.yaml
3.	deployment.yaml	
4.	service.yaml	
5.	migrate-and-colstatic-job.yaml	- Миграции БД и сбор статики в S3	image, command, restartPolicy: Never	
6.	createsuperuser-job.yaml- Создание суперпользователя	image, env (superuser данные)	
7.	cronjob.yaml	- Очистка сессий (ежедневно в 3:00)	schedule: "0 3 * * *", command: clearsessions	

psql-ssl.yaml	- Тестовый под для проверки связи с БД. Подгружать не обязательно

После применения всех манифестов перезапустите django-deployment:

```
kubectl rollout restart deployment django-deployment -n your_namespace
```

#### 6. Nginx и Ingress

nginx-configmap.yaml - Пример настроек Nginx, замените все кластерозависимые параметры в конфигурации на актуальные для вашего окружения, после чего:

1. Удалите дефолтный ConfigMap
```
kubectl delete configmap main-nginx-config -n yournamespace
```

2. Активируйте манифест
```
kubectl apply -f nginx_configmap.yaml -n yournamespace
```

3. ingress_config.txt - Пример настроек Ingress. Аналогично актуализируйте все кластерозависимые параметры, после чего примените настройки.

```
kubectl apply -f ingress_config.txt -n yournamespace
```

4. Перезапустите Nginx
```
kubectl rollout restart deployment main-nginx -n yournamespace
```