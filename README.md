# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


```sh
minikube start
minikube image build -t django_app:latest ./backend_main_django
minikube image ls
minikube addons enable ingress
kubectl apply -f kubernetes/config_map.yaml
kubectl apply -f kubernetes/django.yaml
kubectl get ingress
kubectl get all
minikube tunnel
```


# Деплой в Minikube

1. Установите `minikube`
```sh
brew install minikube
kubectl cluster-info
>>> Kubernetes control plane is running at https://127.0.0.1:58185
>>> CoreDNS is running at https://127.0.0.1:58185/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
>>> To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

2. Соберите образ с приложением `django`
```sh
minikube image build ./backend_main_django/ --image-pull-policy=Never -t django_app
minikube image ls
```

3. Разверните PostgreSQL в кластере
```sh
brew install helm
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install postgres bitnami/postgresql
export POSTGRES_PASSWORD=$(kubectl get secret --namespace default postgres-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
env
kubectl run postgres-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:14.4.0-debian-11-r4 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host postgres-postgresql -U postgres -d postgres -p 5432
```

4. Создайте БД и пользователя в PostgreSQL
```sh
postgres=#
>>> CREATE DATABASE postgres_db;
>>> CREATE USER gennadis WITH ENCRYPTED PASSWORD 'password';
>>> GRANT ALL PRIVILEGES ON DATABASE postgres TO gennadis;
```

5. Примените миграции и создайте суперпользователя
```sh
kubectl get pods
kubectl exec django-deployment-6858fdb4c7-zq4mv -- python manage.py migrate
kubectl exec -it django-deployment-6858fdb4c7-zq4mv -- python manage.py createsuperuser
minikube tunnel
```

6. Создайте `ConfigMap` по образцу
```sh
cp kubernetes/config_map_example.yaml kubernetes/config_map.yaml
nano kubernetes/config_map.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config
data:
  SECRET_KEY: "<SECRET_KEY>"
  DATABASE_URL: 'postgres://POSTGRES_USER:POSTGRES_PASSWORD@HOST_ADDRESS:5432/POSTGRES_DB'
  DEBUG: "True"
  ALLOWED_HOSTS: "127.0.0.1,starburger.test"
```

```sh
kubectl apply -f kubernetes/config_map.yaml
kubectl rollout restart deployment django-deployment
```


7. Активируйте `ingress`
```sh
minikube addons enable ingress
kubectl apply -f kubernetes/django.yaml
kubectl get ingress
```

8. Откройте сайт в браузере [http://127.0.0.1](http://127.0.0.1)
```sh
minikube tunnel
```

9. Активируйте автоматический `clearsessions`
```sh
kubectl apply -f kubernetes/clearsessions.yaml
kubectl create job --from=cronjob/django-clearsessions django-clearsessions-job
```

10. Примените миграции
```sh
kubectl apply -f kubernetes/migrate.yaml
kubectl get pods
kubectl delete job django-migrate-job
```
