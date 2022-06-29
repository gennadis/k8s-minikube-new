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
