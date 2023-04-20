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