### ДЗ по теме Volumes, Storages,StatefulSet

Применим конфигурацию [minio-statefulset.yaml](/kubernetes-networks/minio-statefulset.yaml) и проверим что все поднялось

```
root@ubuntu-otus:~/otus_kuber/lessons-5-kubernetes-volumes# kubectl get pod minio-0
NAME      READY   STATUS    RESTARTS   AGE
minio-0   1/1     Running   0          44s
root@ubuntu-otus:~/otus_kuber/lessons-5-kubernetes-volumes# kubectl get pvc
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-minio-0   Bound    pvc-33e67a53-86e3-41b4-8630-fa63b5ff33d3   10Gi       RWO            standard       49s
root@ubuntu-otus:~/otus_kuber/lessons-5-kubernetes-volumes# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
pvc-33e67a53-86e3-41b4-8630-fa63b5ff33d3   10Gi       RWO            Delete           Bound    default/data-minio-0   standard                56s
```

Для того, чтобы наш StatefulSet был доступен изнутри кластера,создадим Headless Service [minio-headless-service.yaml](/kubernetes-networks/minio-headless-service.yaml) и применим его
```
root@ubuntu-otus:~/otus_kuber/lessons-5-kubernetes-volumes# kubectl apply -f minio-headless-service.yaml
service/minio created
``` 

Создадим ingress для нашего сервиса minio  [ingress-minio.yaml](/kubernetes-networks/ingress-minio) и применим его, после чего провеим что все доступно
```
root@ubuntu-otus:~/otus_kuber/lessons-5-kubernetes-volumes# curl http://192.168.49.2/minio
<?xml version="1.0" encoding="UTF-8"?>
<Error><Code>AccessDenied</Code><Message>Access Denied.</Message><Resource>/</Resource><RequestId>174E67E75EB772C7</RequestId><HostId>b92de9af-a74c-489c-8583-25011f322564</HostId></Error>
```


## Задание со *

Создадим секрет [minio-secret.yaml](/kubernetes-networks/minio-secret.yaml) применим и поправим конфигруцию [minio-statefulset.yaml](/kubernetes-networks/minio-statefulset.yaml)   
чтобы данные для подключения использовались из секрета
```
    spec:
      containers:
      - name: minio
        env:
        - name: MINIO_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: minio-secret
              key: username
        - name: MINIO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: minio-secret
```

И приемним данную конфигурацию:
```
root@ubuntu-otus:~/otus_kuber/lessons-5-kubernetes-volumes# kubectl apply -f minio-secret.yaml
secret/minio-secret created
root@ubuntu-otus:~/otus_kuber/lessons-5-kubernetes-volumes# kubectl apply -f minio-statefulset.yaml
statefulset.apps/minio configured
```

После примения убеждаемся что все ок
```
root@ubuntu-otus:~/otus_kuber/lessons-5-kubernetes-volumes# kubectl get statefulsets
NAME    READY   AGE
minio   1/1     45m
root@ubuntu-otus:~/otus_kuber/lessons-5-kubernetes-volumes# kubectl get pods
NAME                  READY   STATUS    RESTARTS        AGE
minio-0               1/1     Running   0               46m
web-766c75b97-4mqbr   1/1     Running   1 (4h24m ago)   13h
web-766c75b97-6blsb   1/1     Running   1 (4h24m ago)   13h
web-766c75b97-srx55   1/1     Running   1 (4h24m ago)   13h
root@ubuntu-otus:~/otus_kuber/lessons-5-kubernetes-volumes# kubectl get pvc
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-minio-0   Bound    pvc-33e67a53-86e3-41b4-8630-fa63b5ff33d3   10Gi       RWO            standard       46m
root@ubuntu-otus:~/otus_kuber/lessons-5-kubernetes-volumes# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
pvc-33e67a53-86e3-41b4-8630-fa63b5ff33d3   10Gi       RWO            Delete           Bound    default/data-minio-0   standard                46m
```