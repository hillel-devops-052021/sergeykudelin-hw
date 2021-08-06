# Homework: Kubernetes

- Домашнее задание принимается как Pull Request в основной репозиторий
- Имя ветки hw-kubernetes-02
- Наличие всех созданных файлов в каталоге charts
- Описание выполненных действий в README.MD корня репозитория.
- Короткое описание в Description Pull Request-а.
- Reviewer: sergeykudelin


# Создаем Helm Charts для наших компонентов

- Cоздадим новый кластер с помощью minukube
```
➜  sergeykudelin-hw git:(hw-kubernetes-02) ✗ minikube start
😄  minikube v1.22.0 on Darwin 11.4
❗  Both driver=hyperkit and vm-driver=hyperkit have been set.

    Since vm-driver is deprecated, minikube will default to driver=hyperkit.

    If vm-driver is set in the global config, please run "minikube config unset vm-driver" to resolve this warning.

✨  Using the hyperkit driver based on user configuration
👍  Starting control plane node minikube in cluster minikube
🔥  Creating hyperkit VM (CPUs=2, Memory=4000MB, Disk=20000MB) ...
🐳  Preparing Kubernetes v1.21.2 on Docker 20.10.6 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: default-storageclass, storage-provisioner
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```
- Подготовим необходимые плагины (Ingress, CSI driver)

```
➜  charts git:(hw-kubernetes-02) ✗ minikube addons enable ingress
    ▪ Using image docker.io/jettech/kube-webhook-certgen:v1.5.1
    ▪ Using image docker.io/jettech/kube-webhook-certgen:v1.5.1
    ▪ Using image k8s.gcr.io/ingress-nginx/controller:v0.44.0
🔎  Verifying ingress addon...
🌟  The 'ingress' addon is enabled
➜  charts git:(hw-kubernetes-02) ✗ minikube addons enable csi-hostpath-driver
❗  [WARNING] For full functionality, the 'csi-hostpath-driver' addon requires the 'volumesnapshots' addon to be enabled.

You can enable 'volumesnapshots' addon by running: 'minikube addons enable volumesnapshots'

    ▪ Using image k8s.gcr.io/sig-storage/csi-attacher:v3.1.0
    ▪ Using image k8s.gcr.io/sig-storage/csi-external-health-monitor-agent:v0.2.0
    ▪ Using image k8s.gcr.io/sig-storage/csi-node-driver-registrar:v2.0.1
    ▪ Using image k8s.gcr.io/sig-storage/csi-external-health-monitor-controller:v0.2.0
    ▪ Using image k8s.gcr.io/sig-storage/csi-snapshotter:v4.0.0
    ▪ Using image k8s.gcr.io/sig-storage/csi-provisioner:v2.1.0
    ▪ Using image k8s.gcr.io/sig-storage/hostpathplugin:v1.6.0
    ▪ Using image k8s.gcr.io/sig-storage/csi-resizer:v1.1.0
    ▪ Using image k8s.gcr.io/sig-storage/livenessprobe:v2.2.0
🔎  Verifying csi-hostpath-driver addon...
🌟  The 'csi-hostpath-driver' addon is enabled
```
- Актуализируем наш /etc/hosts если адрес кластера поменялся
```
➜  k8s git:(hw-kubernetes-02) ✗ cat /etc/hosts | grep local.io
192.168.64.22 realworld.local.io
192.168.64.22 backend.realworld.local.io
192.168.64.22 adminmongo.realworld.local.io
```

- Создадим каталог для хранения наших будущих Helm Chart-ов с именем "charts" в корне репозитория
```
➜  sergeykudelin-hw git:(hw-kubernetes-02) ✗ mkdir charts   
➜  sergeykudelin-hw git:(hw-kubernetes-02) ✗ tree -L 1      
.
...
├── backend
├── charts
├── docker-compose.dev.yml
├── docker-compose.yml
├── frontend
├── k8s
...
```

## Helm Chart MongoDB

Пожалуй начнем с mongodb, так как backend не сможет запуститься без него.

- Создадим скелет нашего Chart-а.
```
➜  sergeykudelin-hw git:(hw-kubernetes-02) cd charts
➜  charts git:(hw-kubernetes-02) ✗ helm create mongo
Creating mongo
➜  charts git:(hw-kubernetes-02) ✗ tree mongo -L 2
mongo
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
└── values.yaml
```
- Очистим содержимое файла values.yaml
- Удалим все файлы в каталоге mongo/templates
```
➜  charts git:(hw-kubernetes-02) ✗ tree mongo -L 2
mongo
├── Chart.yaml
├── charts
├── templates
└── values.yaml
```
- Перенесем все файлы манифестов для mongodb созданные в прошлой ДЗ, должно получиться следующее
```
➜  charts git:(hw-kubernetes-02) ✗ tree mongo -L 2
mongo
├── Chart.yaml
├── charts
├── templates
│   ├── deployment_admin_mongo.yml
│   ├── ingress_admin_mongo.yml
│   ├── statefulset_mongo.yml
│   ├── svc_admin_mongo.yml
│   └── svc_mongo.yml
└── values.yaml
```
- Сверимся по содержимому файлов
```
➜  charts git:(hw-kubernetes-02) ✗ for i in $(ls mongo/templates/); do echo '\n---file:' $i '\n' && cat mongo/templates/$i; done; 

---file: deployment_admin_mongo.yml 

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin-mongo
  labels:
    app: realworld
    type: admin-mongo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: realworld
      type: admin-mongo
  template:
    metadata:
      labels:
        app: realworld
        type: admin-mongo
    spec:
      containers:
      - name: admin-mongo
        image: adicom/admin-mongo
        ports:
          - containerPort: 8082                               
        imagePullPolicy: Always
        env:
        - name: PORT
          value: "8082"
        - name: DB_HOST
          value: "mongo"


---file: ingress_admin_mongo.yml 

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: admin-mongo
spec:
  rules:
  - host: "adminmongo.realworld.local.io"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: admin-mongo
            port:
              number: 8082

---file: statefulset_mongo.yml 

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo-realworld
  labels:
    app: realworld
    type: mongo
spec:
  serviceName: mongo
  replicas: 1
  selector:
    matchLabels:
      app: realworld
      type: mongo
  template:
    metadata:
      labels:
        app: realworld
        type: mongo
    spec:
      containers:
      - name: mongo-realworld
        image: mongo:latest
        ports:
        - containerPort: 27017                                
        imagePullPolicy: Always
        volumeMounts:
        - name: mongo-pvc-mongo-realworld-0
          mountPath: /data/db
  volumeClaimTemplates:
  - metadata:
      name: mongo-pvc
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: standard

---file: svc_admin_mongo.yml 

---
apiVersion: v1
kind: Service
metadata:
  name: admin-mongo
  labels:
    app: realworld
    type: admin-mongo
spec:
  selector:
    app: realworld
    type: admin-mongo
  ports:
  - protocol: TCP
    port: 8082
    targetPort: 8082

---file: svc_mongo.yml 

apiVersion: v1
kind: Service
metadata:
  name: mongo
  labels:
    app: realworld
    type: mongo
spec:
  ports:
  - port: 27017
    name: mongo
  clusterIP: None
  selector:
    app: realworld
    type: mongo

- Проверим что в нашем кластере ни чего не запущено пока что
```
➜  charts git:(hw-kubernetes-02) ✗ kubectl get pods
No resources found in default namespace.
```

- Проверим версию helm
```
➜  charts git:(hw-kubernetes-02) ✗ helm version  
version.BuildInfo{Version:"v3.6.3", GitCommit:"d506314abfb5d21419df8c7e7e68012379db2354", GitTreeState:"dirty", GoVersion:"go1.16.5"}
```

- Попробуем применить наш первый Helm Chart
```
➜  charts git:(hw-kubernetes-02) ✗ helm install mongo-ns-default ./mongo
NAME: mongo-ns-default
LAST DEPLOYED: Fri Aug  6 23:28:10 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```
- Проверим наличие PV, PVC, Deployment (AdminMongo), StatefulSet (Mongo)
```
➜  charts git:(hw-kubernetes-02) ✗ kubectl get pv                    
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                 STORAGECLASS   REASON   AGE
pvc-86ce6fca-6717-4909-8525-5d58750559a7   1Gi        RWO            Delete           Bound    default/mongo-pvc-mongo-realworld-0   standard                6m29s
➜  charts git:(hw-kubernetes-02) ✗ kubectl get pvc                   
NAME                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongo-pvc-mongo-realworld-0   Bound    pvc-86ce6fca-6717-4909-8525-5d58750559a7   1Gi        RWO            standard       6m37s
➜  charts git:(hw-kubernetes-02) ✗ kubectl get pod
NAME                           READY   STATUS    RESTARTS   AGE
admin-mongo-7655cf9f98-2kn45   1/1     Running   0          3m37s
mongo-realworld-0              1/1     Running   0          3m37s
```
- Посмотрим историю релизов
```
➜  charts git:(hw-kubernetes-02) ✗ helm history mongo-ns-default
REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION     
1               Fri Aug  6 23:34:32 2021        deployed        mongo-0.1.0     1.16.0          Install complete
```
- Посмотрим как эта история реализована в Kubernetes (в качестве secret-ов)
```
➜  charts git:(hw-kubernetes-02) ✗ kubectl get secrets | grep mongo-ns-default
sh.helm.release.v1.mongo-ns-default.v1   helm.sh/release.v1                    1      8m23s
```
- Обновим еще раз наш Release и проверим снова историю Release-ов
```
➜  charts git:(hw-kubernetes-02) ✗ helm upgrade --install mongo-ns-default ./mongo   
Release "mongo-ns-default" has been upgraded. Happy Helming!
NAME: mongo-ns-default
LAST DEPLOYED: Fri Aug  6 23:44:40 2021
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None

➜  charts git:(hw-kubernetes-02) ✗ kubectl get secrets | grep mongo-ns-default    
sh.helm.release.v1.mongo-ns-default.v1   helm.sh/release.v1                    1      10m
sh.helm.release.v1.mongo-ns-default.v2   helm.sh/release.v1                    1      10s
```
- MongoDB готово, двигаемся дальше.

## Backend

- Создадим еще один Chart
```
➜  charts git:(hw-kubernetes-02) ✗ helm create realworld-be
Creating realworld-be
```
- Повторим процедуру удаление всех файлов из каталога с шаблонами и содержимое файла Values
- Перенес манифесты которые относяться к backend-у, должна быть следующая структура
```
➜  charts git:(hw-kubernetes-02) ✗ tree realworld-be -L 2  
realworld-be
├── Chart.yaml
├── charts
├── templates
│   ├── deployment_backend.yml
│   ├── ingress_backend.yml
│   └── svc_backend.yml
└── values.yaml
```
- Содержимое следующее
```
➜  charts git:(hw-kubernetes-02) ✗ for i in $(ls realworld-be/templates/); do echo '\n---file:' $i '\n' && cat realworld-be/templates/$i; done;

---file: deployment_backend.yml 

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: be-realworld
  labels:
    app: realworld
    type: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: realworld
      type: backend
  template:
    metadata:
      labels:
        app: realworld
        type: backend
    spec:
      containers:
      - name: be-realworld
        image: sergeykudelin/hillel-backend:0.0.1
        ports:
          - containerPort: 8081                               
        imagePullPolicy: Always
        livenessProbe:
          httpGet:
            path: /api/status
            port: 8081
          initialDelaySeconds: 30
          timeoutSeconds: 15
        readinessProbe:
          httpGet:
            path: /api/status
            port: 8081
          initialDelaySeconds: 30
          timeoutSeconds: 15
        env:
        - name: PORT
          value: "8081"
        - name: NODE_ENV
          value: "production"
        - name: MONGO_DB_URI
          value: "mongodb://mongo/conduit"
        - name: SECRET
          value: "secret"
---file: ingress_backend.yml 

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: be-realworld
spec:
  rules:
  - host: "backend.realworld.local.io"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: be-realworld
            port:
              number: 8081

---file: svc_backend.yml 

---
apiVersion: v1
kind: Service
metadata:
  name: be-realworld
  labels:
    app: realworld
    type: backend
spec:
  selector:
    app: realworld
    type: backend
  ports:
  - protocol: TCP
    port: 8081
    targetPort: 8081
```
- Применяем наш Helm Chart
```
➜  charts git:(hw-kubernetes-02) ✗ helm install realworld-be-ns-default ./realworld-be
NAME: realworld-be-ns-default
LAST DEPLOYED: Fri Aug  6 23:53:21 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```
- Проверить корректность работы
```
➜  charts git:(hw-kubernetes-02) ✗ helm history realworld-be-ns-default
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION     
1               Fri Aug  6 23:53:21 2021        deployed        realworld-be-0.1.0      1.16.0          Install complete

➜  charts git:(hw-kubernetes-02) ✗ kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
admin-mongo-7655cf9f98-2kn45    1/1     Running   0          20m
be-realworld-6f66f5fc87-c5bpm   1/1     Running   0          118s
be-realworld-6f66f5fc87-cnq7p   1/1     Running   0          118s
mongo-realworld-0               1/1     Running   0          20m

➜  charts git:(hw-kubernetes-02) ✗ kubectl get svc
NAME           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
admin-mongo    ClusterIP   10.110.153.4   <none>        8082/TCP    21m
be-realworld   ClusterIP   10.98.40.110   <none>        8081/TCP    2m38s
kubernetes     ClusterIP   10.96.0.1      <none>        443/TCP     64m
mongo          ClusterIP   None           <none>        27017/TCP   21m

➜  charts git:(hw-kubernetes-02) ✗ kubectl get ingress
NAME           CLASS    HOSTS                           ADDRESS         PORTS   AGE
admin-mongo    <none>   adminmongo.realworld.local.io   192.168.64.22   80      21m
be-realworld   <none>   backend.realworld.local.io      192.168.64.22   80      2m42s
```
- Финально проверим доступность Backend-а для нашей хостовой ОС
```
➜  ~ curl http://backend.realworld.local.io/api/status
{"status":"UP"}
```

## Frontend

- Создаем очередной Helm Chart
```
➜  charts git:(hw-kubernetes-02) ✗ helm create realworld-fe
Creating realworld-fe
```
- Повторим процедуру удаление всех файлов из каталога с шаблонами и содержимое файла Values
- Перенес манифесты которые относяться к frontend-у, должна быть следующая структура
```
➜  charts git:(hw-kubernetes-02) ✗ tree realworld-fe -L 2  
realworld-fe
├── Chart.yaml
├── charts
├── templates
│   ├── deployment_frontend.yml
│   ├── ingress_frontend.yml
│   └── svc_frontend.yml
└── values.yaml
```
- И содержимое файлов
```
➜  charts git:(hw-kubernetes-02) ✗ for i in $(ls realworld-fe/templates/); do echo '\n---file:' $i '\n' && cat realworld-fe/templates/$i; done;

---file: deployment_frontend.yml 

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fe-realworld
  labels:
    app: realworld
    type: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: realworld
      type: frontend
  template:
    metadata:
      labels:
        app: realworld
        type: frontend
    spec:
      containers:
      - name: fe-realworld
        image: sergeykudelin/hillel-frontend:0.0.7
        env:
        - name: REACT_APP_BACKEND_URL
          value: "http://backend.realworld.local.io/api"
        ports:
          - containerPort: 4100                                
        imagePullPolicy: Always

---file: ingress_frontend.yml 

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fe-realworld
spec:
  rules:
  - host: "realworld.local.io"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: fe-realworld
            port: 
              number: 80

---file: svc_frontend.yml 

---
apiVersion: v1
kind: Service
metadata:
  name: fe-realworld
  labels:
    app: realworld
    type: frontend
spec:
  selector:
    app: realworld
    type: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 4100
```
- Применяем и этот Chart
```
➜  charts git:(hw-kubernetes-02) ✗ helm install realworld-fe-ns-default ./realworld-fe
NAME: realworld-fe-ns-default
LAST DEPLOYED: Sat Aug  7 00:04:05 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```
- Проверяем
```
➜  charts git:(hw-kubernetes-02) ✗helm history realworld-fe-ns-default
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION     
1               Sat Aug  7 00:04:05 2021        deployed        realworld-fe-0.1.0      1.16.0          Install complete

➜  charts git:(hw-kubernetes-02) ✗ kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
admin-mongo-7655cf9f98-2kn45    1/1     Running   0          33m
be-realworld-6f66f5fc87-c5bpm   1/1     Running   0          14m
be-realworld-6f66f5fc87-cnq7p   1/1     Running   0          14m
fe-realworld-fdf94f948-4vmwq    1/1     Running   0          3m57s
fe-realworld-fdf94f948-w6lpz    1/1     Running   0          3m56s
mongo-realworld-0               1/1     Running   0          33m
```
- Попробуем обратится с хостой ОС с помощью браузера на страницу регистрации. (НАСТОЯТЕЛЬНО РЕКОММЕНДУЮ ИСПОЛЬЗОВАТЬ INCOGNITO MODE в браузере)

http://realworld.local.io/register

- Убедились что все работает, переходим к следующему пункту. ПОЗДРАВЛЯЮ))) Пол пути пройдено.

- Что дальше? Как не удобно столько всего ставить, но это дает нам гибкость и возможность делегирования над Helm Chart-ами.

## Создаем общий Chart для всего приложения

- Создаем еще один)))
```
➜  charts git:(hw-kubernetes-02) ✗ helm create realworld
Creating realworld
```
- Очистим от лишнего (Ну Вы уже знаете)
- Добавим в данный Chart зависимости в файл Chart.yaml
```
dependencies:
  - name: realworld-be
    version: 0.1.0
    repository: file://../realworld-be
  - name: realworld-fe
    version: 0.1.0
    repository: file://../realworld-fe
```
- Но что бы зависимости могли использоваться нам надо их подтянуть и зафиксировать через digest что бы быть уверенным что не произошла подмена их.
```
➜  charts git:(hw-kubernetes-02) ✗ cd realworld
➜  realworld git:(hw-kubernetes-02) ✗ helm dependency build
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "sergeykudelin" chart repository
...Successfully got an update from the "bitnami" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 2 charts
Deleting outdated charts
```
- Проверим что зависимости подтянулись в каталог Charts и  появился файл Chart.lock
```
➜  realworld git:(hw-kubernetes-02) ✗ tree -L 2
.
├── Chart.lock
├── Chart.yaml
├── charts
│   ├── realworld-be-0.1.0.tgz
│   └── realworld-fe-0.1.0.tgz
├── templates
└── values.yaml
```
- Файл зависимостей сформирован, проверим что на базе него можно получить артефакты
```
➜  realworld git:(hw-kubernetes-02) ✗ rm -rdf ./charts/*.tgz

➜  realworld git:(hw-kubernetes-02) ✗ ll ./charts
total 0
drwxr-xr-x  2 sergeykudelin  staff   64  7 Aug 00:27 .
drwxr-xr-x  8 sergeykudelin  staff  256  7 Aug 00:27 ..

➜  realworld git:(hw-kubernetes-02) ✗ helm dependency update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "sergeykudelin" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 2 charts
Deleting outdated charts

➜  realworld git:(hw-kubernetes-02) ✗ ll ./charts           
total 16
drwxr-xr-x  4 sergeykudelin  staff   128  7 Aug 00:28 .
drwxr-xr-x  8 sergeykudelin  staff   256  7 Aug 00:28 ..
-rw-r--r--  1 sergeykudelin  staff  1119  7 Aug 00:28 realworld-be-0.1.0.tgz
-rw-r--r--  1 sergeykudelin  staff  1016  7 Aug 00:28 realworld-fe-0.1.0.tgz
```
- Есть одна проблема, где в этой цепочке наш Chart с MongoDB? Верно его нет, давайте исправим.
- Добавим следующие строчки в Chart.yaml в Helm Chart для Backend-а
```
dependencies:
  - name: mongo
    version: 0.1.0
    repository: file://../mongo
```
- Должно быть так ...
```
➜  charts git:(hw-kubernetes-02) ✗ cat realworld-be/Chart.yaml | grep -A5 dependencies:
dependencies:
  - name: mongo
    version: 0.1.0
    repository: file://../mongo
```
- Теперь нам надо сформировать файл блокировки и для Chart-а realworld-be
```
➜  charts git:(hw-kubernetes-02) ✗ cd realworld-be
➜  realworld-be git:(hw-kubernetes-02) ✗ helm dependency build
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "sergeykudelin" chart repository
...Successfully got an update from the "bitnami" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 1 charts
Deleting outdated charts
```
- А что же с общий Chart-ом, он уже знает про MongoDB, сомневаюсь, давайте проверим распокавав файл realworld-be-0.1.0.tgz в каталоге ./charts/realworld/charts/, попробуйте найти упоминание mongodb
- Исправим это, перейдем в Chart realworld и пересоберем зависимости
```
➜  realworld git:(hw-kubernetes-02) ✗ helm dependency build
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "sergeykudelin" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 2 charts
Deleting outdated charts
```
- Распакуйте тот же архив еще раз и сравните содержимое. Попробуйте пояснить каким образом helm принял решение загрузить новый архив и почему изменился файл Chart.lock

## Пробуем установить все одним Helm Chart-ом.

- Смотрим что установлено
```
➜  realworld git:(hw-kubernetes-02) ✗ helm list
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
mongo-ns-default        default         2               2021-08-06 23:44:40.098503 +0300 EEST   deployed        mongo-0.1.0             1.16.0     
realworld-be-ns-default default         1               2021-08-06 23:53:21.556454 +0300 EEST   deployed        realworld-be-0.1.0      1.16.0     
realworld-fe-ns-default default         1               2021-08-07 00:04:05.660903 +0300 EEST   deployed        realworld-fe-0.1.0      1.16.0
```
- Удаляем
```
➜  realworld git:(hw-kubernetes-02) ✗ helm uninstall mongo-ns-default
release "mongo-ns-default" uninstalled
➜  realworld git:(hw-kubernetes-02) ✗ helm uninstall realworld-be-ns-default 
release "realworld-be-ns-default" uninstalled
➜  realworld git:(hw-kubernetes-02) ✗ helm uninstall realworld-fe-ns-default 
release "realworld-fe-ns-default" uninstalled
```
- Пробуем поставить одним Chart-ом
```
➜  charts git:(hw-kubernetes-02) ✗ helm install realworld-ns-default ./realworld
NAME: realworld-ns-default
LAST DEPLOYED: Sat Aug  7 00:46:01 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None

➜  charts git:(hw-kubernetes-02) ✗ helm list
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
realworld-ns-default    default         1               2021-08-07 00:46:01.662346 +0300 EEST   deployed        realworld-0.1.0 1.16.0 
```
- Дождитесь успешной установки всех компонентов
```
➜  charts git:(hw-kubernetes-02) ✗ kubectl get pod
NAME                            READY   STATUS    RESTARTS   AGE
admin-mongo-7655cf9f98-98hv4    1/1     Running   0          111s
be-realworld-6f66f5fc87-hjhxk   1/1     Running   1          111s
be-realworld-6f66f5fc87-lhrc6   1/1     Running   1          111s
fe-realworld-fdf94f948-9wmb7    1/1     Running   0          111s
fe-realworld-fdf94f948-qv4nb    1/1     Running   0          111s
mongo-realworld-0               1/1     Running   0          111s
```
- Можете с помощью браузере убедиться в работоспособности приложения.

## Добавим немного шаблонизации, совсем чучуть, но по самому сложному пути с зависимостями.

- Добавим следующий блок в файл values.yaml для realworld-be, то есть будет выглядить так.
```
➜  charts git:(hw-kubernetes-02) ✗ cat realworld-be/values.yaml
image:
  tag: "0.0.1"
```
- Так как мы обновили зависимый Chart, то нам надо снова обновить зависимости для основного Chart-а.
```
➜  charts git:(hw-kubernetes-02) ✗ cd realworld
➜  realworld git:(hw-kubernetes-02) ✗ helm dependency build
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "sergeykudelin" chart repository
...Successfully got an update from the "bitnami" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 2 charts
Deleting outdated charts
```
- Остается вопрос "Как же нам у нас есть менять эти значение из основного Chart-а". Для определения переменных не основого чарта нам достаточно указывать блок со значение в блоке с именем chart-а, то есть ...
```
➜  realworld git:(hw-kubernetes-02) ✗ cd ..
➜  charts git:(hw-kubernetes-02) ✗ cd .
➜  charts git:(hw-kubernetes-02) ✗ cd ..
➜  sergeykudelin-hw git:(hw-kubernetes-02) ✗ cat values.yml 
realworld-be:
  image:
    tag: "0.0.1"
```
- Таким образом мы можем переопределять переменные как в основном Chart-е так в зависимостях. А установка будет выглядить следующим образом.
```
➜  sergeykudelin-hw git:(hw-kubernetes-02) ✗ helm upgrade --install realworld-ns-default -f values.yml ./charts/realworld
Release "realworld-ns-default" has been upgraded. Happy Helming!
NAME: realworld-ns-default
LAST DEPLOYED: Sat Aug  7 01:04:59 2021
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
```
- Проверим что ни чего не поломали
```
➜  sergeykudelin-hw git:(hw-kubernetes-02) ✗ kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
admin-mongo-7655cf9f98-98hv4    1/1     Running   0          15m
be-realworld-6f66f5fc87-hjhxk   1/1     Running   1          15m
be-realworld-6f66f5fc87-lhrc6   1/1     Running   1          15m
fe-realworld-fdf94f948-9wmb7    1/1     Running   0          15m
fe-realworld-fdf94f948-qv4nb    1/1     Running   0          15m
mongo-realworld-0               1/1     Running   0          15m
```

# План минимальный выполнен, дальше пожеланию.

Задание +100 к карме: Попробуйте вынести еще несколько значений в переменные.

Задание +300 к карме: Ознакомьтесь с документацией и сделайте так, что бы Helm Chart можно было применять в разным Namespace-ы