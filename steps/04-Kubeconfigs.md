# Создание `kubeconfigs` для компонентов

Сейчас настало время создать [конфигурационные файлы Kubernetes](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), или `kubeconfigs`, которые позволят клиентам найти и аутентифицироваться в `API-Servers`.

Итак, мы делаем файлы `kubeconfig` для клентов `controller manager`, `kube-proxy`, `scheduler` и для пользователя `admin`.

### Kubernetes Public IP Address

Каждому `kubeconfig` требуется Kubernetes API Server для подключения. Для обеспечения `high availability` мы используем IP, назнаенный для нашего балансировщика. Как помнишь, это у нас `192.168.66.30`.

```
LOADBALANCER_ADDRESS=192.168.66.30
```
### `kubeconfig` для `kubelet`

При создании файлов `kubeconfig` для `kubelet` необходимо использовать сертификат клиента, соответствующий имени узла `kubelet`. Это гарантирует, что `kubelet` будет должным образом авторизован Kubernetes [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/).

*Не забудь запускать команды в тоже же директории, где создал SSL-сертификаты*

Такой файл `kubeconfig` создаем для каждого рабочего узла. У меня он единственный.

Итак, я это делаю на `controlplane01`:
```
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://${LOADBALANCER_ADDRESS}:6443 \
    --kubeconfig=node01.kubeconfig

kubectl config set-credentials system:node:node01 \
    --client-certificate=node01.crt \
    --client-key=node01.key \
    --embed-certs=true \
    --kubeconfig=node01.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:node01 \
    --kubeconfig=node01.kubeconfig

kubectl config use-context default --kubeconfig=node01.kubeconfig
```

Получили:

```
node01.kubeconfig
```



### `kubeconfig` для `kube-proxy`

Создадим `kubeconfig` для службы `kube-proxy`:

```
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://${LOADBALANCER_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.crt \
    --client-key=kube-proxy.key \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

Получили:

```
kube-proxy.kubeconfig
```

Документация по [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)

### `kubeconfig` для `kube-controller-manager`

Создадим `kubeconfig` для службы `kube-controller-manager`:

```
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.crt \
    --client-key=kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

Получили:

```
kube-controller-manager.kubeconfig
```

Документация по [kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)

### `kubeconfig` для `kube-scheduler`

Создадим `kubeconfig` для службы `kube-scheduler`:

```
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.crt \
    --client-key=kube-scheduler.key \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

Получили:

```
kube-scheduler.kubeconfig
```

Документация по [kube-scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)

### `kubeconfig` для пользователя `admin`

Создадим `kubeconfig` для пользователя `admin`:

```
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```

Получили:

```
admin.kubeconfig
```

Справка по [kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)

# Распространение kubeconfigs

Копируем `kube-proxy` конфиг-файл и конфиг для ноды на его воркер:

```
for instance in node01; do
  scp scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:~/
done
```

Копируем `admin.kubeconfig`, `kube-controller-manager` и `kube-scheduler` конфиг-файлы на каждый мастер:

```
for instance in controlplane01 controlplane02; do
  scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${instance}:~/
done
```

Следущий шаг: [Создание ключа для шифрования secrets](steps/05-Encrypt-at-Rest.md)
