### ДЗ по теме Шаблонизация манифестов Kubernetes

ДЗ будет сделано на кластре kubernetes в яндекс обалке.

Ставим Helm 3 версии, проверяем успешность установки:
```
PS C:\Users\kurochkin.k> helm.exe version
version.BuildInfo{Version:"v3.11.2", GitCommit:"912ebc1cd10d38d340f048efaf0abda047c3468e", GitTreeState:"clean", GoVersion:"go1.18.10"}
```

Добавляем репозиторий stable (репо https://kubernetes-charts.storage.googleapis.com больше недоступен,поэтому ставим рекоммендованный репо) и проверяем что все ок:
```
PS C:\Users\kurochkin.k> helm repo add stable https://kubernetes-charts.storage.googleapis.com
Error: repo "https://kubernetes-charts.storage.googleapis.com" is no longer available; try "https://charts.helm.sh/stable" instead

PS C:\Users\kurochkin.k> helm repo add stable "https://charts.helm.sh/stable"
"stable" has been added to your repositories

PS C:\Users\kurochkin.k> helm repo list
NAME    URL
bitnami https://charts.bitnami.com/bitnami
stable  https://charts.helm.sh/stable
```

Создадим namespace и release nginx-ingress

```
kubectl create ns nginx-ingress

PS C:\Users\kurochkin.k> helm repo add nginx-stable https://helm.nginx.com/stable
"nginx-stable" has been added to your repositories
PS C:\Users\kurochkin.k> helm upgrade --install nginx-ingress nginx-stable/nginx-ingress --namespace=nginx-ingress
Release "nginx-ingress" does not exist. Installing it now.
NAME: nginx-ingress
LAST DEPLOYED: Mon Apr  3 10:46:48 2023
NAMESPACE: nginx-ingress
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The NGINX Ingress Controller has been installed.

NAME    NAMESPACE       REVISION        UPDATED STATUS  CHART   APP VERSION
PS C:\Users\kurochkin.k> helm.exe list -a -A
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
nginx-ingress   nginx-ingress   1               2023-04-03 10:56:29.8064023 +0300 MSK   deployed        nginx-ingress-0.17.0    3.1.0
```

##### cert-manager
Ставим cert-manager
```
PS C:\Users\kurochkin.k> helm repo add jetstack https://charts.jetstack.io
"jetstack" has been added to your repositories

PS C:\Users\kurochkin.k> kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.crds.yaml
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io create


PS C:\Users\kurochkin.k> helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.11.0
NAME: cert-manager
LAST DEPLOYED: Mon Apr  3 11:05:10 2023
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
cert-manager v1.11.0 has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://cert-manager.io/docs/configuration/

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://cert-manager.io/docs/usage/ingress/
```

Для корректной работы cert-manager необходим ресурс [ClusterIssuer.yaml](cert-manager/ClusterIssuer.yaml) 

Для того чтобы заработал деплой helm-чарта chartmuseum, необходимо его выкачать и поправить yaml описывающий ресурс ingress, так как мы используем более новую версию кластера кубера:
```
{{- if .Values.ingress.enabled }}
{{- $servicePort := .Values.service.externalPort -}}
{{- $serviceName := include "chartmuseum.fullname" . -}}
{{- $ingressExtraPaths := .Values.ingress.extraPaths -}}
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "chartmuseum.fullname" . }}
  annotations:
{{ toYaml .Values.ingress.annotations | indent 4 }}
  labels:
{{- if .Values.ingress.labels }}
{{ toYaml .Values.ingress.labels | indent 4 }}
{{- end }}
{{ include "chartmuseum.labels.standard" . | indent 4 }}
spec:
  rules:
  {{- range .Values.ingress.hosts }}
  - host: {{ .name }}
    http:
      paths:
      {{- range $ingressExtraPaths }}
      - path: {{ default "/" .path | quote }}
        pathType: Prefix
        backend:
          service:
            name: {{ $.Values.service.servicename }}
            port:
              number: {{ default $servicePort .port }}
      {{- end }}
      - path: {{ default "/" .path | quote }}
        pathType: Prefix
        backend:
          service:
            name: {{ $.Values.service.servicename }}
            name: {{ default $serviceName .service }}
            port:
              number: {{ default $servicePort .servicePort }}
  {{- end }}
  tls:
  {{- range .Values.ingress.hosts }}
  {{- if .tls }}
  - hosts:
    - {{ .name }}
    secretName: {{ .tlsSecret }}
  {{- end }}
  {{- end }}
{{- end -}}
```

После этого деплоим чарт:
```
PS C:\Users\kurochkin.k\Documents\repository_otus\2kw92_platform\kubernetes-templating> helm upgrade --install chartmuseum C:\Users\kurochkin.k\Documents\charts\chartmuseum\ --namespace=chartmuseum -f .\chartmuseum\values.yaml

WARNING: This chart is deprecated
Release "chartmuseum" has been upgraded. Happy Helming!
NAME: chartmuseum
LAST DEPLOYED: Mon Apr  3 13:45:52 2023
NAMESPACE: chartmuseum
STATUS: deployed
REVISION: 3
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **

Get the ChartMuseum URL by running:

  export POD_NAME=$(kubectl get pods --namespace chartmuseum -l "app=chartmuseum" -l "release=chartmuseum" -o jsonpath="{.items[0].metadata.name}")
  echo http://127.0.0.1:8080/
  kubectl port-forward $POD_NAME 8080:8080 --namespace chartmuseum
```

Проверяем с помощью Lens все ли окей:
(/docs/pic/1.PNG)