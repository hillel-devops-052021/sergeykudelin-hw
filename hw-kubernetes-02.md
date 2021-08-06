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
➜  sergeykudelin-hw git:(k8s_intro) ✗ minikube start
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
- Создадим каталог для хранения наших будущих Helm Chart-ов с именем "charts" в корне репозитория
```
➜  sergeykudelin-hw git:(k8s_intro) ✗ mkdir charts   
➜  sergeykudelin-hw git:(k8s_intro) ✗ tree -L 1      
.
├── README.md
├── Vagrantfile
├── backend
├── charts
├── docker-compose.dev.yml
├── docker-compose.yml
├── frontend
├── k8s
└── values.yml
```

## Helm Chart MongoDB

Пожалуй начнем с mongodb, так как backend не сможет запуститься без него.

- Создадим скелет нашего Chart-а.
```
```