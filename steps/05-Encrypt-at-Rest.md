# Создание ключа и конфига для шифрования данных

Kubernetes хранит различные данные, включая состояние кластера, конфигурации приложений и их секреты. Он поддерживает возможность шифровать данные [`at rest`](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data).

Сейчас мы создадим ключ и конфигурацию для [шифрования](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration) `secrets` в Kubernetes.

### Ключ

Создаем ключ шифрования:

```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

### Конфиг

Создаем `encryption-config.yaml` - файл конфигурации шифрования:

```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

### Распространим конфиг

Скопируем `encryption-config.yaml` на каждый узел `controlplane`:

```
for instance in controlplane01 controlplane02; do
  scp encryption-config.yaml ${instance}:~/
done
```

Справка в [документации](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#encrypting-your-data).

Следущий шаг: [Установка etcd](steps/06-ETCD.md)
