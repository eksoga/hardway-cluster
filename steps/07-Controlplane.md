# Установка слоя `Control Plane` на мастерах

### Директория для конфиг-файла

```
sudo mkdir -p /etc/kubernetes/config
```

### Получение исполняемых файлов

Качаем официальный релиз Kubernetes:

```
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl"
```

Справка по [релизам](https://kubernetes.io/docs/setup/release/#server-binaries)

Инсталлируем полученные бинарники:

```
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```

### Настройка `kube-apiserver`

```
sudo mkdir -p /var/lib/kubernetes;
sudo cp ca.crt ca.key kube-apiserver.crt kube-apiserver.key \
    service-account.key service-account.crt \
    etcd-server.key etcd-server.crt \
    encryption-config.yaml /var/lib/kubernetes/
```

Внутренний IP адрес будет использован для анонсирования API Server для участников кластера. Получим его командой:

```
INTERNAL_IP=$(ip addr show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f 1)
```

Проверим, все ли установилось:

```
echo $INTERNAL_IP
```

Создадим unit-файл для службы systemd `kube-apiserver.service`:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=2 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.crt \\
  --enable-admission-plugins=NodeRestriction,ServiceAccount \\
  --enable-bootstrap-token-auth=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.crt \\
  --etcd-certfile=/var/lib/kubernetes/etcd-server.crt \\
  --etcd-keyfile=/var/lib/kubernetes/etcd-server.key \\
  --etcd-servers=https://192.168.66.11:2379,https://192.168.66.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.crt \\
  --kubelet-client-certificate=/var/lib/kubernetes/kube-apiserver.crt \\
  --kubelet-client-key=/var/lib/kubernetes/kube-apiserver.key \\
  --runtime-config=api/all=true \\
  --service-account-key-file=/var/lib/kubernetes/service-account.crt \\
  --service-cluster-ip-range=10.96.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kube-apiserver.crt \\
  --tls-private-key-file=/var/lib/kubernetes/kube-apiserver.key \\
  --service-account-issuer=${INTERNAL_IP} \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account.key \\
  --v=10
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Настройка `kube-controller-manager`

Копируем `kube-controller-manager` конфиг в нужное место:

```
sudo cp kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

Создадим unit-файл для службы systemd `kube-controller-manager.service`:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=192.168.66.0/24 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.crt \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca.key \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.crt \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account.key \\
  --service-cluster-ip-range=10.96.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Настройка `kube-scheduler`

Копируем `kube-scheduler` конфиг в нужное место:

```
sudo cp kube-scheduler.kubeconfig /var/lib/kubernetes/
```

Создадим unit-файл для службы systemd `kube-scheduler.service`:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig \\
  --address=127.0.0.1 \\
  --leader-elect=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Запустим службы `controlplane`

```
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

*Дай 15-20 секунд для kube-apiserver, чтобы запуститься*



# RBAC для авторизации `kubelet`

Нам нужно настроить разрешения RBAC, чтобы позволить `kube-apiserver.service` получать доступ к `kubelet` на каждом рабочем узле. Доступ к `kubelet API` необходим для получения метрик, логов и выполнения команд в PODs.

*Здесь мы устанавливаем флаг kubelet `--authorization-mode` на `Webhook`. В режиме веб-хука для определения авторизации используется API [SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access).*

Команды в этом разделе будут влиять на весь кластер, и их нужно будет запустить только один раз с одного из мастеров.

Создадим `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) с разрешениями `kubelet API` и выполнять наиболее распространенные задачи, связанные с управлением PODs:

```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

Сервер API Kubernetes аутентифицируется в `kubelet` как пользователь `kubernetes`, используя сертификат клиента, что определено флагом` --kubelet-client-certificate`.

Привяжем ClusterRole `system: kube-apiserver-to-kubelet` к пользователю `kubernetes`:

```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

# Проверка

```
kubectl cluster-info --kubeconfig admin.kubeconfig
```
Получим:
```
Kubernetes control plane is running at https://127.0.0.1:6443
```

*Не забудь сделать это на всех мастерах: `controlplane01` и `controlplane02`*

```
curl  https://192.168.66.30:6443/version -k
```

Получим:

```
{
  "major": "1",
  "minor": "21",
  "gitVersion": "v1.21.0",
  "gitCommit": "cb303e613a121a29364f75cc67d3d580833a7479",
  "gitTreeState": "clean",
  "buildDate": "2021-04-08T16:25:06Z",
  "goVersion": "go1.16.1",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Следущий шаг: [Установка worker](steps/08-Workers.md)

