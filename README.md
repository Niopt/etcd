# Etcd
**Основная идея** 
- Манифест в Kubernetes — это YAML-файл с описанием ресурса.
- Когда ты применяешь его через kubectl apply -f ..., Kubernetes API Server превращает его в объект API и сохраняет в виде ключ-значение в etcd.
- etcd не хранит YAML напрямую, а хранит сериализованный объект (обычно в JSON или protobuf).

**Пример манифеста**

```apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
spec:
  containers:
    - name: nginx
      image: nginx:latest
```

**Как это выглядит в etcd**

Ключ:

`/registry/pods/default/nginx`

Значение (JSON, protobuf) примерно такое:

```{
  "metadata": {
    "name": "nginx",
    "namespace": "default",
    "creationTimestamp": "2025-10-19T16:00:00Z",
    "labels": {}
  },
  "spec": {
    "containers": [
      {"name": "nginx", "image": "nginx:latest"}
    ]
  },
  "status": {
    "phase": "Running",
    "hostIP": "10.0.0.1",
    "podIP": "10.244.0.2"
  }
}
```

- metadata — данные о ресурсе (имя, namespace, метки, ревизия)
- spec — то, что описано в манифесте
- status — состояние, которое обновляется контроллерами (например, Pod был создан и запущен на ноде)

**Иерархия ключей**

- Все манифесты хранятся под префиксом /registry:
  `/registry/pods/<namespace>/<pod-name>`
  `/registry/deployments/<namespace>/<deployment-name>`
  `/registry/services/<namespace>/<service-name>`
- Такая структура позволяет API Server делать watch на префикс и уведомлять контроллеры о любых изменениях.

**Ревизии и автокомпактация**

- Каждый апдейт манифеста создаёт новую ревизию ключа в etcd.
- Автокомпактация (auto-compaction) удаляет старые версии, чтобы база не росла.
- Это позволяет откатить объект к предыдущей версии через API.

**Ключи в etcd и JSON манифест**

Когда ты видишь структуру вроде:

```/registry
    /nodes
        /node1
```

- Это ключи в etcd — путь, по которому хранится объект.
- Значение ключа — это сериализованный объект, который обычно JSON или protobuf.

То есть /registry/nodes/node1 это ключ, а сам объект ноды — значение.

# Пример

Для node1 значение в etcd может быть примерно таким (JSON):

```{
  "metadata": {
    "name": "node1",
    "labels": {"role": "worker"},
    "creationTimestamp": "2025-10-19T16:00:00Z"
  },
  "status": {
    "capacity": {"cpu": "4", "memory": "8Gi"},
    "conditions": [{"type": "Ready", "status": "True"}]
  }
}
```

- metadata — имя, метки, время создания
- status — текущее состояние ноды


**В чём разница с YAML манифестом?**

- YAML манифест — это файл, который подаёшь через kubectl.
- Etcd хранит этот объект уже сериализованным, плюс сгенерированные поля (например status, resourceVersion).
- Пример: манифест Pod:

```apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
spec:
  containers:
    - name: nginx
      image: nginx:latest
```

в etcd будет храниться ключ:
`/registry/pods/default/nginx`

со значением:

```{
  "metadata": {...},
  "spec": {...},
  "status": {...}
}
```

То есть ключи в etcd — это путь, а JSON — это содержимое объекта.

# Создание 

Для начала надо создать ключи и сертификаты для tls соединена.
В папке /etc/etcd/ssl создаем

**СА для подписи**

- CA будет подписывать все сертификаты:
`openssl genrsa -out ca.key 4096`
- Генерация самоподписанного сертификата CA
`openssl req -x509 -new -nodes -key ca.key -subj "/CN=etcd-ca" -days 3650 -out ca.crt`

**Сертификат сервера (для client-transport-security) чтобы клиенты могли подключаться через tls**

- создаём ключ и CSR
`openssl genrsa -out etcd-server.key 2048`
`openssl req -new -key etcd-server.key -subj "/CN=etcd-server" -out etcd-server.csr`
- подписываем сертификат через CA

```openssl x509 -req -in etcd-server.csr -CA ca.crt -CAkey ca.key -CAcreateserial   -out etcd-server.crt -days 3650   -extensions v3_req -extfile <(echo "
[ v3_req ]
subjectAltName = @alt_names
[ alt_names ]
DNS.1 = localhost
IP.1 = 127.0.0.1
IP.2 = 127.0.0.2
IP.3 = 127.0.0.3
")
```

IP.1 = 127.0.0.1 - указыват для каких адресов этот сертификат

**Сертификат peer-узлов (для peer-transport-security) чтобы другие etcd могли работать в кластере через tls**

Для каждого узла (например, node1, node2, node3) нужно создать отдельный ключ и сертификат, чтобы узлы могли аутентифицироваться друг перед другом.

**Node_1**

- ключ и CSR для node1
`openssl genrsa -out etcd-peer-node1.key 2048`
`openssl req -new -key etcd-peer-node1.key -subj "/CN=etcd-peer-node1" -out etcd-peer-node1.csr`
- подписываем сертификат через CA

```openssl x509 -req -in etcd-peer-node1.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out etcd-peer-node1.crt -days 3650 \
  -extensions v3_req -extfile <(echo "
[ v3_req ]
subjectAltName = @alt_names
[ alt_names ]
DNS.1 = localhost
IP.1 = 127.0.0.1
IP.2 = 127.0.0.2
IP.3 = 127.0.0.3
")
```

**Node_2**

- ключ и CSR для node2
`openssl genrsa -out etcd-peer-node2.key 2048`
`openssl req -new -key etcd-peer-node2.key -subj "/CN=etcd-peer-node2" -out etcd-peer-node2.csr`
- подписываем сертификат через CA

```openssl x509 -req -in etcd-peer-node2.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out etcd-peer-node2.crt -days 3650 \
  -extensions v3_req -extfile <(echo "
[ v3_req ]
subjectAltName = @alt_names
[ alt_names ]
DNS.1 = localhost
IP.1 = 127.0.0.2
IP.2 = 127.0.0.1
IP.3 = 127.0.0.3
")
```

**Node_3**
- ключ и CSR для node3
`openssl genrsa -out etcd-peer-node3.key 2048`
`openssl req -new -key etcd-peer-node3.key -subj "/CN=etcd-peer-node3" -out etcd-peer-node3.csr`
- подписываем сертификат через CA

```openssl x509 -req -in etcd-peer-node3.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out etcd-peer-node3.crt -days 3650 \
  -extensions v3_req -extfile <(echo "
[ v3_req ]
subjectAltName = @alt_names
[ alt_names ]
DNS.1 = localhost
IP.1 = 127.0.0.3
IP.2 = 127.0.0.1
IP.3 = 127.0.0.2
")
```

Далее нам надо создать директории для самих хранилищ
`mkdir -p /var/lib/etcd/{node1,node2,node3}`

Далее нам надо создать файлы для Log 
`touch /var/log/{etcd_1.log,etcd_2.log,etcd_3.log}`

далее запускаем все три хранилища

`etcd --config-file etcd_1.conf.yml`
`etcd --config-file etcd_2.conf.yml`
`etcd --config-file etcd_3.conf.yml`

проверям роботоспособность 

```etcdctl --endpoints=https://127.0.0.1:2381,https://127.0.0.2:2383,https://127.0.0.3:2385 \
  --cacert=/etc/etcd/ssl/ca.crt \
  --cert=/etc/etcd/ssl/etcd-server.crt \
  --key=/etc/etcd/ssl/etcd-server.key \
  member list
```
