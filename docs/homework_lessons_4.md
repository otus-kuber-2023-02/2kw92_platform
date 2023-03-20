### ДЗ по теме Сетевое взаимодействие Pod,сервисы

##### Добавление проверок Pod

Добавляем в описание пода web-pod.yaml readinessProbe
```
apiVersion: v1 # Версия API
kind: Pod # Объект, который создаем
metadata:
 name: myweb
 labels: # Метки в формате key: value
   app: web
spec: # Описание Pod
 containers: # Описание контейнеров внутри Pod
 - name: web
   image: 2kw92/homework_docker:homework_v1
   readinessProbe: # Добавим проверку готовности
     httpGet: # веб-сервера отдавать
       path: /index.html # контент
       port: 80
```
 
Далее запускаем под и првоеряем его состояние:
```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks# kubectl apply -f web-pod.yaml
pod/myweb created
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks# kubectl get pod myweb
NAME    READY   STATUS    RESTARTS   AGE
myweb   0/1     Running   0          27s
```

Смотрим список Conditions:
```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks# kubectl describe pod myweb | grep Conditions -A 5
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
```

Смотрим на список событий, связанных с Pod:
```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks# kubectl describe pod myweb | grep War
  Warning  Unhealthy  50s (x38 over 5m53s)  kubelet            Readiness probe failed: Get "http://10.244.0.3:80/index.html": dial tcp 10.244.0.3:80: connect: connection refused
```

Из листинга выше видно, что проверка готовности контейнера завершается неудачно. Это неудивительно - веб-сервер в контейнере слушает порт 8000 (по условиям первого ДЗ).  
Пока мы не будем исправлять эту ошибку, а добавим другой вид проверок:
```
    livenessProbe:
      tcpSocket:
        port: 8000
```
Получаем опсиание пода:
```
apiVersion: v1 # Версия API
kind: Pod # Объект, который создаем
metadata:
 name: myweb
 labels: # Метки в формате key: value
   app: web
spec: # Описание Pod
 containers: # Описание контейнеров внутри Pod
 - name: web
   image: 2kw92/homework_docker:homework_v1
   readinessProbe: # Добавим проверку готовности
     httpGet: # веб-сервера отдавать
       path: /index.html # контент
       port: 80
   livenessProbe:
      tcpSocket:
        port: 8000
```

И запускаем под с новой конфигурацией:
```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks# kubectl apply -f web-pod.yaml --force
pod/myweb unchanged
```

##### Вопрос для самопроверки:

1. Почему следующая конфигурация валидна, но не имеет смысла?
```
livenessProbe:
  exec:
    command:
      - 'sh'
      - '-c'
      - 'ps aux | grep my_web_server_process'
```
**Ответ:Команда всегда выполнится, так как всегда будет найден процесс запустивший grep. Например:**
```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks# kubectl exec -i -t  myweb sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
/ $ ps aux | grep ngdasdas
   48 nginx     0:00 grep ngdasdas
```

2. Бывают ли ситуации, когда она все-таки имеет смысл?  
**Если ее немного доработать, убрав из вывода строку grep,  на случай если протсо нужно найти запущенный процесс, например:**
```
/ $ ps aux | grep process | grep -v grep
    1 nginx     0:00 nginx: master process nginx -g daemon off;
    6 nginx     0:00 nginx: worker process
```


##### Создание Deployment:
Теперь создадим Deployment [web-pod.yaml](/kubernetes-networks/web-pod.yaml) и применем его предварительно удалив старый под и посмотрим что получилось:
```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl delete pod/myweb --grace-period=0 --force
Warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "myweb" force deleted

root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl apply -f web-pod.yaml
deployment.apps/web created

root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl describe deploy/web | grep Cond -A 6
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      False   MinimumReplicasUnavailable
  Progressing    True    ReplicaSetUpdated
OldReplicaSets:  <none>
NewReplicaSet:   web-5597d5bb6f (3/3 replicas created)
```

Поскольку мы не исправили `ReadinessProbe` , то поды, входящие в наш Deployment, не переходят в состояние *Ready* из-за неуспешной проверки.  
На предыдущем листинге видно, что это влияет на состояние всего Deployment (строчка Available в блоке Conditions).  
Теперь самое время исправить ошибку! Поменяем в файле [web-pod.yaml](/kubernetes-networks/web-pod.yaml) следующие параметры:
* Увеличим число реплик до 3 ( replicas: 3 )
* Исправим порт в readinessProbe на порт 8000

И применем изменения:
```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl apply -f web-pod.yaml
deployment.apps/web configured
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl describe deploy/web | grep Cond -A 6
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   web-766c75b97 (3/3 replicas created)
```
Видим что все ок.  

Протестируем работу блока strategy. Для этого в описание [web-pod.yaml](/kubernetes-networks/web-pod.yaml) добавим блок:
```
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 100%
```

**maxUnavailable**— это необязательное поле, в котором указывается максимальное количество подов, которые могут быть недоступны в процессе обновления
**maxSurge**— это необязательное поле, в котором указывается максимальное количество модулей Pod, которое может быть создано сверх желаемого количества модулей Pod

Результаты наблюдений:

**maxUnavailable: 100%**
**maxSurge: 100%**
```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl get pod
NAME                   READY   STATUS        RESTARTS   AGE
web-54df57b6c6-fvk9x   1/1     Terminating   0          5m57s
web-54df57b6c6-hsgv4   1/1     Terminating   0          5m57s
web-54df57b6c6-wjnxp   1/1     Terminating   0          5m57s
web-65c66dcb5f-8mtdh   0/1     Init:0/1      0          4s
web-65c66dcb5f-pl562   0/1     Init:0/1      0          4s
web-65c66dcb5f-stsv9   0/1     Init:0/1      0          4s
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl get pod
NAME                   READY   STATUS     RESTARTS   AGE
web-65c66dcb5f-8mtdh   0/1     Init:0/1   0          6s
web-65c66dcb5f-pl562   0/1     Init:0/1   0          6s
web-65c66dcb5f-stsv9   0/1     Init:0/1   0          6s
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl get pod
NAME                   READY   STATUS            RESTARTS   AGE
web-65c66dcb5f-8mtdh   0/1     Init:0/1          0          8s
web-65c66dcb5f-pl562   0/1     PodInitializing   0          8s
web-65c66dcb5f-stsv9   0/1     Init:0/1          0          8s
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl get pod
NAME                   READY   STATUS    RESTARTS   AGE
web-65c66dcb5f-8mtdh   1/1     Running   0          19s
web-65c66dcb5f-pl562   1/1     Running   0          19s
web-65c66dcb5f-stsv9   1/1     Running   0          19s
```
Видим что одновремено удаляются все старые поды и создаются все новые поды. При таком обновления есть недоступность системы

**maxUnavailable: 0**
**maxSurge: 0**
```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl apply -f web-pod.yaml
The Deployment "web" is invalid: spec.strategy.rollingUpdate.maxUnavailable: Invalid value: intstr.IntOrString{Type:0, IntVal:0, StrVal:""}: may not be 0 when `maxSurge` is 0
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# vi web-pod.yaml
```
Поймали ошибку что максимальное количество подов, которые могут быть недотсупны не может быть равно 0 

**maxUnavailable: 0**
**maxSurge: 100%**
```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl get po
NAME                   READY   STATUS     RESTARTS   AGE
web-5bc7c79bbd-9qkl2   0/1     Init:0/1   0          7s
web-5bc7c79bbd-g8x4x   0/1     Running    0          7s
web-5bc7c79bbd-z7hs6   0/1     Init:0/1   0          7s
web-65c66dcb5f-8mtdh   1/1     Running    0          4m31s
web-65c66dcb5f-pl562   1/1     Running    0          4m31s
web-65c66dcb5f-stsv9   1/1     Running    0          4m31s
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl get po
NAME                   READY   STATUS        RESTARTS   AGE
web-5bc7c79bbd-9qkl2   0/1     Init:0/1      0          9s
web-5bc7c79bbd-g8x4x   1/1     Running       0          9s
web-5bc7c79bbd-z7hs6   0/1     Init:0/1      0          9s
web-65c66dcb5f-8mtdh   1/1     Running       0          4m33s
web-65c66dcb5f-pl562   1/1     Terminating   0          4m33s
web-65c66dcb5f-stsv9   1/1     Running       0          4m33s
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl get po
NAME                   READY   STATUS        RESTARTS   AGE
web-5bc7c79bbd-9qkl2   0/1     Init:0/1      0          14s
web-5bc7c79bbd-g8x4x   1/1     Running       0          14s
web-5bc7c79bbd-z7hs6   1/1     Running       0          14s
web-65c66dcb5f-8mtdh   1/1     Terminating   0          4m38s
web-65c66dcb5f-stsv9   1/1     Running       0          4m38s
```
Старые поды удаляются, только после того как новые перешли в активное состояние.