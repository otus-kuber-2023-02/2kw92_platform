## ДЗ по теме Операторы, CustomResourceDefinition

ДЗ будет сделано на кластре kubernetes в яндекс облаке.

Cоздадим CustomResource [cr.yml](/kubernetes-operators/deploy/cr.yml)
И примеенем его:
```
PS C:\Users\kurochkin.k\Documents\repository_otus\2kw92_platform\kubernetes-operators\deploy> kubectl.exe apply -f .\cr.yml
error: resource mapping not found for name: "mysql-instance" namespace: "" from ".\\cr.yml": no matches for kind "MySQL" in version "otus.homework/v1"
ensure CRDs are installed first
```

Создадим CRD [crd.yml](/kubernetes-operators/deploy/crd.yml) и применем его:
```
PS C:\Users\kurochkin.k\Documents\repository_otus\2kw92_platform\kubernetes-operators\deploy> kubectl.exe apply -f crd.yml --validate=false
customresourcedefinition.apiextensions.k8s.io/mysqls.otus.homework created
```

И применим CR:
```
PS C:\Users\kurochkin.k\Documents\repository_otus\2kw92_platform\kubernetes-operators\deploy> kubectl.exe apply -f .\cr.yml --validate=false
mysql.otus.homework/mysql-instance create
```

Повзаимодействуем с созданными объектами:
```
PS C:\Users\kurochkin.k\Documents\repository_otus\2kw92_platform\kubernetes-operators\deploy> kubectl.exe get crd
NAME                                             CREATED AT
apdoslogconfs.appprotectdos.f5.com               2023-04-03T07:46:45Z
apdospolicies.appprotectdos.f5.com               2023-04-03T07:46:45Z
aplogconfs.appprotect.f5.com                     2023-04-03T07:46:44Z
appolicies.appprotect.f5.com                     2023-04-03T07:46:44Z
apusersigs.appprotect.f5.com                     2023-04-03T07:46:45Z
certificaterequests.cert-manager.io              2023-04-03T08:03:04Z
certificates.cert-manager.io                     2023-04-03T08:03:04Z
challenges.acme.cert-manager.io                  2023-04-03T08:03:03Z
clusterissuers.cert-manager.io                   2023-04-03T08:03:02Z
dnsendpoints.externaldns.nginx.org               2023-04-03T07:46:45Z
dosprotectedresources.appprotectdos.f5.com       2023-04-03T07:46:45Z
globalconfigurations.k8s.nginx.org               2023-04-03T07:46:45Z
issuers.cert-manager.io                          2023-04-03T08:03:04Z
mysqls.otus.homework                             2023-04-20T14:05:06Z
orders.acme.cert-manager.io                      2023-04-03T08:03:05Z
policies.k8s.nginx.org                           2023-04-03T07:46:46Z
transportservers.k8s.nginx.org                   2023-04-03T07:46:46Z
virtualserverroutes.k8s.nginx.org                2023-04-03T07:46:46Z
virtualservers.k8s.nginx.org                     2023-04-03T07:46:46Z
volumesnapshotclasses.snapshot.storage.k8s.io    2023-03-15T14:43:03Z
volumesnapshotcontents.snapshot.storage.k8s.io   2023-03-15T14:43:04Z
volumesnapshots.snapshot.storage.k8s.io          2023-03-15T14:43:04Z
PS C:\Users\kurochkin.k\Documents\repository_otus\2kw92_platform\kubernetes-operators\deploy> kubectl get mysqls.otus.homework
NAME             AGE
mysql-instance   3m19s
PS C:\Users\kurochkin.k\Documents\repository_otus\2kw92_platform\kubernetes-operators\deploy> kubectl describe mysqls.otus.homework mysql-instance
Name:         mysql-instance
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  otus.homework/v1
Kind:         MySQL
Metadata:
  Creation Timestamp:  2023-04-20T14:05:34Z
  Generation:          1
  Managed Fields:
    API Version:  otus.homework/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        .:
        f:image:
    Manager:         kubectl-client-side-apply
    Operation:       Update
    Time:            2023-04-20T14:05:34Z
  Resource Version:  228885
  UID:               dad90dfd-55c4-4a5a-b9c1-ead9ab2c04c9
Spec:
  Image:  mysql:5.7
Events:   <none>
```

Провалидируем наши cr и сrd, добавив необходимые поля  для объектов типа mysql и применим манифесты без ключа --validate=false
```
PS C:\Users\kurochkin.k\Documents\repository_otus\2kw92_platform\kubernetes-operators\deploy> kubectl.exe apply -f crd.yml
customresourcedefinition.apiextensions.k8s.io/mysqls.otus.homework unchanged
PS C:\Users\kurochkin.k\Documents\repository_otus\2kw92_platform\kubernetes-operators\deploy> kubectl.exe apply -f cr.yml
mysql.otus.homework/mysql-instance created
```

Добавим описание обязательных полей с помощью блока
```
              required:
                - image
                - database
                - password
                - storage_size  
```
И попробуем применим CR без одного из полей, получим ошибку:
```
PS C:\Users\kurochkin.k\Documents\repository_otus\2kw92_platform\kubernetes-operators\deploy> kubectl.exe apply -f crd.yml
customresourcedefinition.apiextensions.k8s.io/mysqls.otus.homework configured
PS C:\Users\kurochkin.k\Documents\repository_otus\2kw92_platform\kubernetes-operators\deploy> kubectl.exe apply -f cr.yml
error: error validating "cr.yml": error validating data: ValidationError(MySQL.spec): missing required field "database" in homework.otus.v1.MySQL.spec; if you choose to ignore these errors, turn validation off with --validate=false
```

### Деплой оператора

Примените манифесты:

[service-account.yml](/kubernetes-operators/deploy/service-account.yml)
[role.yml](/kubernetes-operators/deploy/role.yml)
[role-binding.yml](/kubernetes-operators/deploy/role-binding.yml)
[deploy-operator.yml](/kubernetes-operators/deploy/deploy-operator.yml)

```
PS C:\Users\kurochkin.k\Documents\repository_otus\2kw92_platform\kubernetes-operators\deploy> kubectl.exe apply -f .\service-account.yml
serviceaccount/mysql-operator created
PS C:\Users\kurochkin.k\Documents\repository_otus\2kw92_platform\kubernetes-operators\deploy> kubectl.exe apply -f .\role.yml
clusterrole.rbac.authorization.k8s.io/mysql-operator created
PS C:\Users\kurochkin.k\Documents\repository_otus\2kw92_platform\kubernetes-operators\deploy> kubectl.exe apply -f .\role-binding.yml
clusterrolebinding.rbac.authorization.k8s.io/workshop-operator created
PS C:\Users\kurochkin.k\Documents\repository_otus\2kw92_platform\kubernetes-operators\deploy> kubectl.exe apply -f .\deploy-operator.yml
deployment.apps/mysql-operator created
```
Создаем CR (если еще не создан):

```
PS C:\Users\kurochkin.k\Documents\repository_otus\2kw92_platform\kubernetes-operators\deploy> kubectl apply -f .\cr.yml
mysql.otus.homework/mysql-instance created
```

Проверим, что все работает:

```
PS C:\Users\kurochkin.k\Documents\repository_otus\2kw92_platform\kubernetes-operators\deploy> kubectl.exe get pvc
NAME                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
backup-mysql-instance-pvc   Bound    pvc-7a0789ed-f0a3-43a7-a6c7-53db0fee656b   1Gi        RWO            yc-network-hdd   2m6s
mysql-instance-pvc          Bound    pvc-2579d872-5f15-47af-90a6-0ebf5807c378   1Gi        RWO            yc-network-hdd   2m6s

root@ubuntu-otus:~# export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
root@ubuntu-otus:~# kubectl exec -it $MYSQLPOD -- mysql -u root -potuspassword -e "CREATE TABLE test ( id smallint unsigned not null auto_increment, name varchar(20) not null, constraint pk_example primary key
>  (id) );" otus-database
mysql: [Warning] Using a password on the command line interface can be insecure.
root@ubuntu-otus:~# kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) VALUES (
> null, 'some data' );" otus-database
mysql: [Warning] Using a password on the command line interface can be insecure.
root@ubuntu-otus:~# kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) VALUES (
> null, 'some data-2' );" otus-database
mysql: [Warning] Using a password on the command line interface can be insecure.
root@ubuntu-otus:~# kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-------------+
| id | name        |
+----+-------------+
|  1 | some data   |
|  2 | some data-2 |
+----+-------------+
```

Далее удалим инстанс и заново создадим:

```
root@ubuntu-otus:~# kubectl delete mysqls.otus.homework mysql-instance
mysql.otus.homework "mysql-instance" deleted

root@ubuntu-otus:~# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                               STORAGECLASS     REASON   AGE
backup-mysql-instance-pv                   1Gi        RWO            Retain           Available                                                                 5m18s
pvc-2579d872-5f15-47af-90a6-0ebf5807c378   1Gi        RWO            Delete           Released    default/mysql-instance-pvc          yc-network-hdd            5m11s
pvc-7a0789ed-f0a3-43a7-a6c7-53db0fee656b   1Gi        RWO            Delete           Bound       default/backup-mysql-instance-pvc   yc-network-hdd            5m12s

root@ubuntu-otus:~/otus_kuber/lessons-8-kubernetes-operators/deploy# kubectl apply -f cr.yml
mysql.otus.homework/mysql-instance created

root@ubuntu-otus:~/otus_kuber/lessons-8-kubernetes-operators/deploy# kubectl exec -it mysql-instance-55bdd6bdd6-t4zzg -- mysql -potuspassword -e "select * from test;" otus-database
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-------------+
| id | name        |
+----+-------------+
|  1 | some data   |
|  2 | some data-2 |
+----+-------------+

```

Видим что данный восстановились и джобы успешно отработали
```
root@ubuntu-otus:~/otus_kuber/lessons-8-kubernetes-operators/deploy# kubectl get jobs
NAME                         COMPLETIONS   DURATION   AGE
backup-mysql-instance-job    1/1           4s         5m55s
restore-mysql-instance-job   1/1           61s        3m45s
```