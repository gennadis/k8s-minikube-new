1. Разверните сайт на своей машине
```sh
docker compose up -d --build
```
[http://127.0.0.1:8080/admin/](http://127.0.0.1:8080/admin/)

2. Переключите сайт в production-режим  
`DEBUG: ${WEB_DEBUG-FALSE}`

3. Поднимите кластер Minikube
```
brew install minikube
kubectl cluster-info
```

4. Запустите в кластере веб-сервер
```sh
kubectl run nginx --image=nginx:1.23.0-alpine --port=80
kubectl port-forward pod/nginx 8000:80
```

5. Загрузите образ Django в кластер
```sh
minikube image build ./backend_main_django/ --image-pull-policy=Never -t django_app
minikube image ls
kubectl run django --image=django_app --image-pull-policy=Never --env="SECRET_KEY=<SECRET_KEY>"
kubectl exec django -ti -- python manage.py shell
```

6. Запустите базу данных снаружи кластера
```sh
kubectl run django --image=django_app --image-pull-policy=Never --env="SECRET_KEY=<SECRET_KEY>" --env="DATABASE_URL=postgres://test_k8s:OwOtBep9Frut@<HOST_ADDRESS>:5432/test_k8s"  # set your database `HOST_ADDRESS`
kubectl exec django -ti -- python manage.py migrate
```

```sh
kubectl exec django -ti -- python manage.py shell
>>> from django.contrib.auth.models import User
>>> User.objects.all()
<QuerySet [<User: root>]>
```

7. Запустите сайт Django
```sh
kubectl port-forward pod/django 8000:80
```
[http://127.0.0.1:8080/admin/](http://127.0.0.1:8080/admin/)

8. Опубликуйте сайт
```sh
kubectl create deployment django-deployment --image=django_app
kubectl expose deployment django-deployment --type=NodePort --port=80
```

10. Настройте Deployment
```sh
kubectl apply -f kubernetes/django.yaml
minikube service django-service --url
```

11. Вынесите переменные окружения из конфигурационных файлов
```sh
mv ./kubernetes/config_map_example.yaml ./kubernetes/config_map.yaml
kubectl apply -f kubernetes/config_map.yaml
kubectl rollout restart deployment django-deployment
minikube service django-service --url
```
12. Настройте Ingress
```sh
minikube addons enable ingress
kubectl apply -f kubernetes/django.yaml
kubectl get ingress
minikube tunnel
```

13. Настройте регулярное удаление сессий
```sh
kubectl apply -f kubernetes/clearsessions.yaml
kubectl create job --from=cronjob/django-clearsessions django-clearsessions-job
```

14. Упакуйте команду migrate
```sh
kubectl apply -f kubernetes/migrate.yaml
kubectl get pods
kubectl logs django-migrate-xsgk7
```
