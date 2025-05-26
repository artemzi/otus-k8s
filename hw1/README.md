Подготовка

```bash
$ minikube start --memory=4096 --cpus=2 --driver=docker --cni=bridge --insecure-registry true

$ minikube addons enable metrics-server
$ minikube addons enable storage-provisioner

$ docker build --no-cache -t django-app .

$ eval $(minikube docker-env)
```

Когда нужно отдать окружение

```bash
$ eval $(minikube docker-env -u)
```

Применяем конфиг

```bash
$ kubectl apply -f k8s/postgres-secret.yaml
$ kubectl apply -f k8s/postgres-statefulset.yaml
$ kubectl apply -f k8s/pvc.yaml
$ kubectl apply -f k8s/postgres-service.yaml
$ kubectl apply -f k8s/deployment.yaml
```

Запуск приложения

```bash
$ minikube dashboard &

$ kubectl port-forward deployment/django-app 8000:8000
```

Проверка

```bash
$ curl -I http://localhost:8000
```

MinIO Operator

0) https://min.io/docs/minio/kubernetes/upstream/operations/installation.html
1) https://min.io/docs/minio/kubernetes/upstream/operations/cert-manager.html#minio-certmanager
2) https://min.io/docs/minio/kubernetes/upstream/operations/cert-manager/cert-manager-operator.html#minio-certmanager-operator

```bash
$ kubectl apply -k "github.com/minio/operator?ref=v5.0.18"
$ kubectl scale deployment --namespace minio-operator minio-operator --replicas 1

$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.13/cert-manager.yaml
$ kubectl apply -f k8s/selfsigned-root-clusterissuer.yaml
$ kubectl apply -f k8s/operator-ca-tls-secret.yaml
$ kubectl apply -f k8s/sts-tls-certificate.yaml
```

Add tenants for MinIO

```bash
$ helm repo add minio-operator https://operator.min.io

$ curl -sLo values.yaml https://raw.githubusercontent.com/minio/operator/master/helm/tenant/values.yaml

$ helm install \
--namespace minio-operator \
--create-namespace \
--values k8s/values.yaml \
minio-operator minio-operator/tenant

$ kubectl port-forward svc/myminio-hl 9000 -n minio-operator # можно забить и ходить по 9443 но в целом редирект работает
$ kubectl port-forward svc/myminio-console 9443 -n minio-operator

```

Настройка Velero для резервного копирования
секреты так хранить не нужно, но с этим сетапом мы ничего не боимся...

```bash
$ helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
$ helm repo update

$ kubectl create namespace velero

$ kubectl create secret generic velero-credentials \
  --from-file=cloud=/Users/alfa/Workspace/otus-k8s/hw1/k8s/credentials-velero \
  -n velero

$ helm install velero vmware-tanzu/velero \
  --namespace velero \
  --create-namespace \
  --values k8s/velero-values.yaml

# Пример создания резервной копии PostgreSQL StatefulSet
$ kubectl -n velero create -f - <<EOF
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: postgres-backup
  namespace: velero
spec:
  includedNamespaces:
  - default
  includedResources:
  - statefulsets
  - persistentvolumes
  - persistentvolumeclaims
  - secrets
  - configmaps
  - services
EOF

# Проверка статуса backup
$ velero backup get

# расписание
$ velero schedule create postgres-daily --schedule="@daily" --include-namespaces default
```

Для восстановления из резервной копии:

```bash
# Восстановление из backup
$ velero restore create --from-backup postgres-backup

# мониторинг
$ kubectl -n velero port-forward deployment/velero 8085:8085
```