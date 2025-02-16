# Ставим кластер `ETCD`

Компоненты Kubernetes не имеют состояния - stateless - и хранят состояние кластера в [базе](https://github.com/etcd-io/etcd) `ETCD`. 

Сейчас мы поднимем двухузловой кластер `ETCD` и настроим его для обеспечения высокой доступности и безопасного доступа.

### Запуск команд в параллели

Я использую терминал `MobaXterm`, в нем можно разделить экраны и работать сразу на нескольких машинах.
Утилиты вроде [tmux](https://github.com/tmux/tmux/wiki) позволяют запускать одинаковые команды на нескольких узлах одновременно.

### Download and Install the etcd Binaries

Качаем официальный релиз `ETCD` из [etcd-io](https://github.com/etcd-io/etcd) GitHub project:

```
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.15/etcd-v3.4.15-linux-amd64.tar.gz"
```

Распакуем и установим сервер `etcd` и утилиту управления `etcdctl`:

```
tar -xvf etcd-v3.4.15-linux-amd64.tar.gz
sudo mv etcd-v3.4.15-linux-amd64/etcd* /usr/local/bin/
```

### Настройка сервера

```
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo chmod 700 /var/lib/etcd
sudo cp ca.crt etcd-server.key etcd-server.crt /etc/etcd/
```
Внутренний IP-адрес ноды будет использоваться для обслуживания клиентских запросов и связи с узлами etcd-кластера. Узнаем  внутренние адреса etcd-узлов (у нас совпадают с мастерами):
```
INTERNAL_IP=$(ip addr show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f 1)
```

Каждый участник etcd-кластера должен иметь уникальное имя. Зададим имена для них в соответствии с hostname ноды:

```
ETCD_NAME=$(hostname -s)
```

Создадим unit-файл для службы systemd `etcd.service`:

```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/etcd-server.crt \\
  --key-file=/etc/etcd/etcd-server.key \\
  --peer-cert-file=/etc/etcd/etcd-server.crt \\
  --peer-key-file=/etc/etcd/etcd-server.key \\
  --trusted-ca-file=/etc/etcd/ca.crt \\
  --peer-trusted-ca-file=/etc/etcd/ca.crt \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controlplane01=https://192.168.66.11:2380,controlplane02=https://192.168.66.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Запустим сервер

```
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

*Не забудь, что это мы делаем на всех мастерах: `controlplane01` и `controlplane02`*

# Проверка

Список участников кластера `ETCD`:

```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/etcd-server.crt \
  --key=/etc/etcd/etcd-server.key
```

Получили:

```
afa765ec329b3a69, started, controlplane02, https://192.168.66.12:2380, https://192.168.66.12:2379, false
e68096fcbeeca62f, started, controlplane01, https://192.168.66.11:2380, https://192.168.66.11:2379, false
```

Справка из [документации](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#starting-etcd-clusters)

Следущий шаг: [Установка controlplane](steps/07-Controlplane.md)
