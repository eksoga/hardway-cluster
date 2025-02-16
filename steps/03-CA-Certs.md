# Создание CA и сертификатов

Здесь мы созадем [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure), используя `openssl`.
С ее помощью мы сделаем Certificate Authority и сгенерируем TLS-сертификаты для компонентов Kubernetes:
- etcd
- kube-apiserver
- kube-controller-manager
- kube-scheduler
- kubelet
- kube-proxy

### Где создавать

В любом месте с установленым `openssl`. Но имей в виду, что тебе потребуется скопировать созданные файлы на VMs. Или делай как я на `controlplane01`.

### Certificate Authority

Созданим центр сертификации, который будет удостоверять остальные сертификаты.

Создаем ключ, запрос на подписание и самоподписанный сертификат для CA:

```
# Create private key for CA
openssl genrsa -out ca.key 2048

# Comment line starting with RANDFILE in /etc/ssl/openssl.cnf definition to avoid permission issues
sudo sed -i '0,/RANDFILE/{s/RANDFILE/\#&/}' /etc/ssl/openssl.cnf

# Create CSR using the private key
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr

# Self sign the csr using its own private key
openssl x509 -req -in ca.csr -signkey ca.key -CAcreateserial  -out ca.crt -days 1000
```
Получили:

```
ca.crt
ca.key
```

Подробнее можно посмотреть в [certificates](https://kubernetes.io/docs/tasks/administer-cluster/certificates/#openssl).

`ca.crt` - это сертификат центра сертификации Kubernetes, а ca.key - закрытый ключ центра сертификации Kubernetes.
Мы будем использовать файл ca.crt во многих местах, поэтому он будет часто копироваться.

`ca.key` используется центром сертификации для подписания сертификатов и должен находиться в надежном месте.

В моем случае `controlplane01` является также `CA`-сервером кластера, поэтому мы будем хранить ключ там. Т.е. у меня нет необходимости его куда-то копировать.

### Клиентские и серверные сертификаты

Здесь мы будем создавать сертификаты клиентов и серверов для каждого компонента Kubernetes, а также клиентский сертификат для пользователя `admin`.

### Клиентский сертификат для пользователя `admin`

Создадим ключ и клиентский сертификат для `admin`:

```
# Generate private key for admin user
openssl genrsa -out admin.key 2048

# Generate CSR for admin user. Note the OU.
openssl req -new -key admin.key -subj "/CN=admin/O=system:masters" -out admin.csr

# Sign certificate for admin user using CA servers private key
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out admin.crt -days 1000
```

*Обрати внимание, что `admin` часть группы **system:masters**. Это дает ему возможность для осуществления административных операций в кластере*

Получили:

```
admin.key
admin.crt
```

*`admin.crt` и `admin.key` мы используем для настройки утилиты `kubectl`, чтобы реализовать административные функции в Kubernetes*

### Клиентский сертификат для `kubelet`

Kubernetes использует специальный режим авторизации под названием `Node Authorizer`, который специально авторизует запросы API, сделанные `kubelet`. Чтобы авторизоваться с помощью `Node Authorizer`, `kubelet` должен использовать учетные данные, которые идентифицируют его как принадлежащего к группе `system: nodes`, с именем пользователя `system:node:<nodeName>`. 

В этом разделе мы создадем сертификат для каждого рабочего узла Kubernetes, который соответствует требованиям `Node Authorizer`.

Команда `openssl` не принимает альтернативные имена в качестве параметра командной строки. Поэтому мы создаем `conf` файл:

```
cat > openssl-pi-worker-01.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = pi-worker-01.local
IP.1 = 192.168.66.111
EOF
```

Создадим сертификат и закрытый ключ для нашего единственного рабочего узла Kubernetes:

```
openssl genrsa -out pi-worker-01.key 2048 
openssl req -new -key pi-worker-01.key -subj "/CN=system:node:pi-worker-01.local/O=system:nodes" -out pi-worker-01.csr -config openssl-pi-worker-01.cnf 
openssl x509 -req -in pi-worker-01.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out pi-worker-01.crt -extensions v3_req -extfile openssl-pi-worker-01.cnf -days 3650
```

Получили:

```
node01.key
node01.crt
```

### Клиентский сертификат для `controller-manager`

Создаем закрытый ключ и сертификат для `kube-controller-manager`:

```
openssl genrsa -out kube-controller-manager.key 2048
openssl req -new -key kube-controller-manager.key -subj "/CN=system:kube-controller-manager" -out kube-controller-manager.csr
openssl x509 -req -in kube-controller-manager.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-controller-manager.crt -days 3650
```

Получили:

```
kube-controller-manager.key
kube-controller-manager.crt
```


### Клиентский сертификат для `kube-proxy`

Создаем закрытый ключ и сертификат для `kube-proxy`:


```
openssl genrsa -out kube-proxy.key 2048
openssl req -new -key kube-proxy.key -subj "/CN=system:kube-proxy" -out kube-proxy.csr
openssl x509 -req -in kube-proxy.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-proxy.crt -days 3650
```

Получили:

```
kube-proxy.key
kube-proxy.crt
```

### Клиентский сертификат для `kube-scheduler`

Создаем закрытый ключ и сертификат для `kube-scheduler`:


```
openssl genrsa -out kube-scheduler.key 2048
openssl req -new -key kube-scheduler.key -subj "/CN=system:kube-scheduler" -out kube-scheduler.csr
openssl x509 -req -in kube-scheduler.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-scheduler.crt -days 3650
```

Получили:

```
kube-scheduler.key
kube-scheduler.crt
```

### Серверный сертификат для `kube-apiserver`

Сертификату для `kube-apiserver` требуется, чтобы все имена, которые могут быть получены различными компонентами, 
были частью альтернативных имен. К ним относятся различные имена DNS и IP-адреса, такие как IP-адрес мастера, 
IP-адрес балансировщика нагрузки, IP-адрес службы kube-api и т. д.

Как и раньше, для сообщения альтернативных имен создаем `conf` файл:

```
cat > openssl.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1
IP.2 = 192.168.88.101
IP.3 = 192.168.88.1
IP.4 = 127.0.0.1
EOF
```

Создаем ключ и сертификат:

```
openssl genrsa -out kube-apiserver.key 2048
openssl req -new -key kube-apiserver.key -subj "/CN=kubernetes" -out kube-apiserver.csr -config openssl.cnf
openssl x509 -req -in kube-apiserver.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-apiserver.crt -extensions v3_req -extfile openssl.cnf -days 3650
```

Получили:

```
kube-apiserver.crt
kube-apiserver.key
```
*Серверу Kubernetes API автоматически назначается внутреннее DNS-имя kubernetes, которое будет связано с первым IP-адресом (10.96.0.1) из диапазона адресов (10.96.0.0/24), зарезервированного для внутренних служб кластера, это мы настроим в главе установки узлов controlplane*

### Серверный сертификат для `ETCD`

Также как и ранее, в сертификате нужно указать все части ETCD-кластера.

Сначала `conf` файл:

```
cat > openssl-etcd.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
IP.1 = 192.168.88.101
IP.2 = 127.0.0.1
EOF
```

Теперь сертификат `ETCD`:

```
openssl genrsa -out etcd-server.key 2048
openssl req -new -key etcd-server.key -subj "/CN=etcd-server" -out etcd-server.csr -config openssl-etcd.cnf
openssl x509 -req -in etcd-server.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out etcd-server.crt -extensions v3_req -extfile openssl-etcd.cnf -days 3650
```

Получили:

```
etcd-server.key
etcd-server.crt
```

### `Key Pair` для `Service Account`

Kubernetes `Controller Manager` использует пару ключей для генерации и подписи токенов `serviceaccounts`. Посмотри в документации, в разделе [managing service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/).

Создаем ключ и сертификат:

```
openssl genrsa -out service-account.key 2048
openssl req -new -key service-account.key -subj "/CN=service-accounts" -out service-account.csr
openssl x509 -req -in service-account.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out service-account.crt -days 3650
```

Получили:

```
service-account.key
service-account.crt
```


# Распространение сертификатов

Скопируем нужные сертификаты и закрытые ключи на `controlplane`-ноды:

```
for instance in controlplane01 controlplane02; do
  scp ca.crt ca.key kube-apiserver.key kube-apiserver.crt \
    service-account.key service-account.crt \
    etcd-server.key etcd-server.crt \
    ${instance}:~/
done
```

Тоже самое для рабочих нод:

```
for instance in node01; do
  scp ca.crt ${instance}.key ${instance}.crt ${instance}:~/
done
```

*Клиентские сертификаты для `kube-proxy`, `kube-controller-manager`, `kube-scheduler`, и `kubelet` 
будут использованы для создания конфигурационных файлов в следующей главе. Эти сертификаты нужно внести в файлы конфигурации для аутентификации клиента. Затем мы скопируем эти файлы конфигурации на другие мастера*


Следущий шаг: [Создание kubeconfigs для компонентов](steps/04-Kubeconfigs.md)
