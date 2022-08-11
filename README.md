# Домашнее задание к занятию "12.2 Команды для работы с Kubernetes"

## Задание 1: Запуск пода из образа в деплойменте

```
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ kubectl create deployment hello-node-deployment --image=k8s.gcr.io/echoserver:1.4 --replicas=2
deployment.apps/hello-node-deployment created
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ kubectl get deployment
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
hello-node-deployment   2/2     2            2           14s
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
hello-node-deployment-574c5cbcb5-gmvtq   1/1     Running   0          28s
hello-node-deployment-574c5cbcb5-pvwxv   1/1     Running   0          28s
vagrant@vagrant:~/netology-12-kubernetes-02-commands$
```

## Задание 2: Просмотр логов для разработки

1. Создание клиентского сертификаты.   
```
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ mkdir cert && cd cert
vagrant@vagrant:~/netology-12-kubernetes-02-commands/cert$ openssl genrsa -out user1.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
..+++++
.......+++++
e is 65537 (0x010001)
vagrant@vagrant:~/netology-12-kubernetes-02-commands/cert$ openssl req -new -key user1.key -out user1.csr -subj "/CN=user1"
vagrant@vagrant:~/netology-12-kubernetes-02-commands/cert$ openssl x509 -req -in user1.csr -CA ~/.minikube/ca.crt -CAkey ~/.minikube/ca.key -CAcreateserial -out user1.crt -days 500
Signature ok
subject=CN = user1
Getting CA Private Key
vagrant@vagrant:~/netology-12-kubernetes-02-commands/cert$
```

2. Создание пользователя.   
```
vagrant@vagrant:~/netology-12-kubernetes-02-commands/cert$ kubectl config set-credentials user1 --client-certificate=user1.crt --client-key=user1.key
User "user1" set.
vagrant@vagrant:~/netology-12-kubernetes-02-commands/cert$ kubectl config set-context user1-context --cluster=minikube --namespace=app-namespace  --user=user1
Context "user1-context" created.
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /home/vagrant/.minikube/ca.crt
    extensions:
    - extension:
        last-update: Wed, 10 Aug 2022 16:36:39 MSK
        provider: minikube.sigs.k8s.io
        version: v1.26.1
      name: cluster_info
    server: https://192.168.49.2:8443
  name: minikube
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://158.160.3.47:8443
  name: mk
contexts:
- context:
    cluster: mk
    user: kubernetes-admin
  name: kubernetes-admin@mk
- context:
    cluster: minikube
    extensions:
    - extension:
        last-update: Wed, 10 Aug 2022 16:36:39 MSK
        provider: minikube.sigs.k8s.io
        version: v1.26.1
      name: context_info
    namespace: default
    user: minikube
  name: minikube
- context:
    cluster: minikube
    namespace: app-namespace
    user: user1
  name: user1-context
current-context: user1-context
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: minikube
  user:
    client-certificate: /home/vagrant/.minikube/profiles/minikube/client.crt
    client-key: /home/vagrant/.minikube/profiles/minikube/client.key
- name: user1
  user:
    client-certificate: /home/vagrant/netology-12-kubernetes-02-commands/cert/user1.crt
    client-key: /home/vagrant/netology-12-kubernetes-02-commands/cert/user1.key
vagrant@vagrant:~/netology-12-kubernetes-02-commands/cert$ kubectl config use-context user1-context
Switched to context "user1-context".
vagrant@vagrant:~/netology-12-kubernetes-02-commands/cert$ kubectl config current-context
user1-context
vagrant@vagrant:~/netology-12-kubernetes-02-commands/cert$ kubectl create namespace app-namespace
Error from server (Forbidden): namespaces is forbidden: User "user1" cannot create resource "namespaces" in API group "" at the cluster scope
```

3. Предоставление доступа пользователю, через создание роли и привязки роли.

```
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ cat role.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: app-namespace
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ cat role-binding.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: app-namespace
subjects:
- kind: User
  name: user1
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ kubectl config use-context minikube
Switched to context "minikube".
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ kubectl apply -f role.yaml
role.rbac.authorization.k8s.io/pod-reader configured
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ kubectl apply -f role-binding.yaml
rolebinding.rbac.authorization.k8s.io/read-pods unchanged
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ kubectl get roles
NAME         CREATED AT
pod-reader   2022-08-11T17:38:15Z
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ kubectl get rolebindings
NAME        ROLE              AGE
read-pods   Role/pod-reader   40m
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ kubectl config use-context user1-context
Switched to context "user1-context".
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ kubectl get pods -n=app-namespace
NAME                                     READY   STATUS    RESTARTS   AGE
hello-node-deployment-574c5cbcb5-tx8t5   1/1     Running   0          23m
hello-node-deployment-574c5cbcb5-xj9j2   1/1     Running   0          23m
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ kubectl create namespace app-namespace
Error from server (Forbidden): namespaces is forbidden: User "user1" cannot create resource "namespaces" in API group "" at the cluster scope
vagrant@vagrant:~/netology-12-kubernetes-02-commands$
```

## Задание 3: Изменение количества реплик

```
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ kubectl config use-context minikube
Switched to context "minikube".
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ kubectl get deploy
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
hello-node-deployment   2/2     2            2           3d
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ kubectl scale --replicas=5 deployment hello-node-deployment
deployment.apps/hello-node-deployment scaled
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ kubectl get deploy
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
hello-node-deployment   5/5     5            5           3d
vagrant@vagrant:~/netology-12-kubernetes-02-commands$ kubectl get pods
NAME                                     READY   STATUS    RESTARTS      AGE
hello-node-deployment-574c5cbcb5-27k7z   1/1     Running   0             22s
hello-node-deployment-574c5cbcb5-gmvtq   1/1     Running   3 (28h ago)   3d
hello-node-deployment-574c5cbcb5-pvwxv   1/1     Running   2 (28h ago)   3d
hello-node-deployment-574c5cbcb5-s9qgx   1/1     Running   0             22s
hello-node-deployment-574c5cbcb5-z5vkt   1/1     Running   0             22s
vagrant@vagrant:~/netology-12-kubernetes-02-commands$
```
