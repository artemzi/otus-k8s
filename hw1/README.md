Подготовка

```
minikube start --insecure-registry true

minikube addons enable metrics-server
minikube addons enable storage-provisioner

docker build --no-cache -t django-app .
eval $(minikube docker-env)
```

Применяем конфиг

```
kubectl apply -f k8s/postgres-statefulset.yaml
kubectl apply -f k8s/postgres-secret.yaml
kubectl apply -f k8s/postgres-service.yaml
kubectl apply -f k8s/pvc.yaml
kubectl apply -f k8s/deployment.yaml
```

Запуск приложения

```
kubectl port-forward deployment/django-app 8000:8000
```

Проверка

```
curl -I http://localhost:8000
```

Отдать окружение

```
eval $(minikube docker-env -u)
```