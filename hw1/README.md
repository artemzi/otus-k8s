Подготовка

```
$ minikube start --memory=4096 --cpus=2 --driver=docker --cni=bridge --insecure-registry true

$ minikube addons enable metrics-server
$ minikube addons enable storage-provisioner

$ docker build --no-cache -t django-app .

$ eval $(minikube docker-env)
```

Когда нужно отдать окружение

```
$ eval $(minikube docker-env -u)
```

Применяем конфиг

```
$ kubectl apply -f k8s/postgres-secret.yaml
$ kubectl apply -f k8s/postgres-statefulset.yaml
$ kubectl apply -f k8s/pvc.yaml
$ kubectl apply -f k8s/postgres-service.yaml
$ kubectl apply -f k8s/deployment.yaml
```

Запуск приложения

```
$ minikube dashboard &

$ kubectl port-forward deployment/django-app 8000:8000
```

Проверка

```
$ curl -I http://localhost:8000
```

MinIO Operator

0) https://min.io/docs/minio/kubernetes/upstream/operations/installation.html
1) https://min.io/docs/minio/kubernetes/upstream/operations/cert-manager.html#minio-certmanager
2) https://min.io/docs/minio/kubernetes/upstream/operations/cert-manager/cert-manager-operator.html#minio-certmanager-operator

```
$ kubectl apply -k "github.com/minio/operator?ref=v5.0.18"
$ kubectl scale deployment --namespace minio-operator minio-operator --replicas 1

$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.13/cert-manager.yaml
$ kubectl apply -f k8s/selfsigned-root-clusterissuer.yaml
$ kubectl apply -f k8s/operator-ca-tls-secret.yaml
$ kubectl apply -f k8s/sts-tls-certificate.yaml
```

Add tenants for MinIO

```
$ helm repo add minio-operator https://operator.min.io

$ curl -sLo values.yaml https://raw.githubusercontent.com/minio/operator/master/helm/tenant/values.yaml

$ helm install \
--namespace minio-operator \
--create-namespace \
--values k8s/values.yaml \
minio-operator minio-operator/tenant

$ kubectl port-forward svc/myminio-hl 9000 -n minio-operator

```