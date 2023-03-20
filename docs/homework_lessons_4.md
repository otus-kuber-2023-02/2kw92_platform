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


##### Создание Service | ClusterIP

Cоздадим манифест для нашего сервиса [web-pod.yaml](/kubernetes-networks/web-svc-cip.yaml) и применим его:
```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl apply -f web-svc-cip.yaml
service/web-svc-cip created
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl get svc
NAME          TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
kubernetes    ClusterIP   10.96.0.1     <none>        443/TCP   118m
web-svc-cip   ClusterIP   10.96.42.21   <none>        80/TCP    5s
```

Подключимся к ВМ Minikube (команда minikube ssh и затем sudo-i ):
Сделайте `curl http://10.96.42.21/index.html` - работает!
Сделайте `ping 10.96.42.21` - пинга нет
Сделайте `arp -an , ip addr show` - нигде нет ClusterIP
Сделайте `iptables --list -nv -t nat` - вот где наш кластерный IP!


```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# minikube ssh

docker@minikube:~$ curl http://10.96.42.21/index.html
<html>
<head>
<title>       </title>
<style type="text/css">
<!--
h1      {text-align:center;
        font-family:Arial, Helvetica, Sans-Serif;
        }

p       {text-indent:20px;
        }
-->
</style>
</head>
<body bgcolor = "#ffffcc" text = "#000000">
<h1>Vpered Loko!!!</h1>

</body>


docker@minikube:~$ ping 10.96.42.21
PING 10.96.42.21 (10.96.42.21) 56(84) bytes of data.
^Z
[1]+  Stopped                 ping 10.96.42.21

root@minikube:~# iptables --list -nv -t nat | grep 10.96.42.21
    1    60 KUBE-SVC-6CZTMAROCN3AQODZ  tcp  --  *      *       0.0.0.0/0            10.96.42.21          /* default/web-svc-cip cluster IP */ tcp dpt:80
    1    60 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.96.42.21          /* default/web-svc-cip cluster IP */ tcp dpt:80
```

##### Включение IPVS

Включим IPVS для kube-proxy , исправив ConfigMap (конфигурация Pod, хранящаяся в кластере).
В  файле конфигурации kube-proxy правим параметры:
```

```
 ipvs:
   strictARP: true
 mode: "ipvs"
```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl --namespace kube-system edit configmaps kube-proxy
configmap/kube-proxy edited
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl --namespace kube-system delete pod --selector='k8s-app=kube-proxy'
pod "kube-proxy-dptrx" deleted
```

Теперь нужно почистить мусор:
```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks/kubernetes-networks# kubectl --namespace kube-system exec kube-proxy-nstqp -it sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
kube-proxy --cleanup
I0320 12:01:00.488605    1464 server.go:224] "Warning, all flags other than --config, --write-config-to, and --cleanup are deprecated, please begin using a config file ASAP"
```
Полностью очистим все правила iptables
Создадим в ВМ с Minikube файл /tmp/iptables.cleanup:
```
*nat
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
COMMIT
*filter
COMMIT
*mangle
COMMIT
```

Теперь применим конфиг и проверим результат:
```
root@minikube:~# iptables-restore /tmp/iptables.cleanup
root@minikube:~# iptables --list -nv -t nat
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 18 packets, 1080 bytes)
 pkts bytes target     prot opt in     out     source               destination
   18  1080 KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

Chain POSTROUTING (policy ACCEPT 18 packets, 1080 bytes)
 pkts bytes target     prot opt in     out     source               destination
   18  1080 KUBE-POSTROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */
    0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0

Chain KUBE-LOAD-BALANCER (0 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain KUBE-MARK-MASQ (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK or 0x4000

Chain KUBE-NODE-PORT (1 references)
 pkts bytes target     prot opt in     out     source               destination

Chain KUBE-POSTROUTING (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* Kubernetes endpoints dst ip:port, source ip for solving hairpin purpose */ match-set KUBE-LOOP-BACK dst,dst,src
   18  1080 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            mark match ! 0x4000/0x4000
    0     0 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK xor 0x4000
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service traffic requiring SNAT */ random-fully

Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination
    2   120 RETURN     all  --  *      *       127.0.0.0/8          0.0.0.0/0
    0     0 KUBE-MARK-MASQ  all  --  *      *      !10.244.0.0/16        0.0.0.0/0            /* Kubernetes service cluster ip + port for masquerade purpose */ match-set KUBE-CLUSTER-IP dst,dst
    8   480 KUBE-NODE-PORT  all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            match-set KUBE-CLUSTER-IP dst,dst
```

Далее поставим ipvadm на машине с minikube и проверим наш кластерный Ip

```
root@minikube:~# ipvsadm --list -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 192.168.49.2:8443            Masq    1      2          0
TCP  10.96.0.10:53 rr
  -> 10.244.0.9:53                Masq    1      0          0
TCP  10.96.0.10:9153 rr
  -> 10.244.0.9:9153              Masq    1      0          0
TCP  10.103.73.43:80 rr
  -> 10.244.0.7:8000              Masq    1      0          0
  -> 10.244.0.8:8000              Masq    1      0          0
  -> 10.244.0.10:8000             Masq    1      0          0
UDP  10.96.0.10:53 rr
  -> 10.244.0.9:53                Masq    1      0          0
```

##### Установка MetalLB
Так как версия MetalLB указанная в д/з устарела ставлю поновее
```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks# kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.5/config/manifests/metallb-native.yaml
namespace/metallb-system created
customresourcedefinition.apiextensions.k8s.io/addresspools.metallb.io configured
customresourcedefinition.apiextensions.k8s.io/bfdprofiles.metallb.io configured
customresourcedefinition.apiextensions.k8s.io/bgpadvertisements.metallb.io configured
customresourcedefinition.apiextensions.k8s.io/bgppeers.metallb.io configured
customresourcedefinition.apiextensions.k8s.io/communities.metallb.io configured
customresourcedefinition.apiextensions.k8s.io/ipaddresspools.metallb.io configured
customresourcedefinition.apiextensions.k8s.io/l2advertisements.metallb.io configured
serviceaccount/controller created
serviceaccount/speaker created
role.rbac.authorization.k8s.io/controller created
role.rbac.authorization.k8s.io/pod-lister created
clusterrole.rbac.authorization.k8s.io/metallb-system:controller configured
clusterrole.rbac.authorization.k8s.io/metallb-system:speaker configured
rolebinding.rbac.authorization.k8s.io/controller created
rolebinding.rbac.authorization.k8s.io/pod-lister created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:controller unchanged
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:speaker unchanged
secret/webhook-server-cert created
service/webhook-service created
deployment.apps/controller created
daemonset.apps/speaker created
validatingwebhookconfiguration.admissionregistration.k8s.io/metallb-webhook-configuration configured
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks# kubectl --namespace metallb-system get all
NAME                              READY   STATUS    RESTARTS   AGE
pod/controller-54dbb75479-lxr68   1/1     Running   0          63s
pod/speaker-5cfbl                 1/1     Running   0          62s

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/webhook-service   ClusterIP   10.105.253.169   <none>        443/TCP   63s

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/speaker   1         1         1       1            1           kubernetes.io/os=linux   63s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   1/1     1            1           63s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-54dbb75479   1         1         1       63s
```
Видим что все ок поднялось и работает.

Создайте манифест [metallb-config.yaml](/kubernetes-networks/metallb-config.yaml) и применим его:
```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks# kubectl apply -f metallb-config.yaml
configmap/config created
```

Далее применяем [web-svc-lb.yaml](/kubernetes-networks/web-svc-lb.yaml).
И Видим что `EXTERNAL-IP` в состоянии `pendind`
```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks# kubectl get svc web-svc-lb
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
web-svc-lb   LoadBalancer   10.105.82.67   <pending>     80:30566/TCP   11m
```

Вопрос почему так происходит, все сделано по инструкции из д.з

Заработает только после создания объекта 
```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
```
И его применения:
```
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks# kubectl apply -f ipaddresspool.yaml
ipaddresspool.metallb.io/first-pool created
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks# kubectl ger svc
error: unknown command "ger" for "kubectl"

Did you mean this?
        set
        get
root@ubuntu-otus:~/otus_kuber/lessons-4-kubernetes-networks# kubectl get svc
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1      <none>          443/TCP        9m47s
web-svc-lb   LoadBalancer   10.99.54.186   192.168.1.240   80:31556/TCP   5m15s
```