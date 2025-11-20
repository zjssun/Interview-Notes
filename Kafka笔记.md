#### dokcer启动kafka(自动安装)
```bash
docker run -d --name kafka `
  -p 9092:9092 `
  -v "D:\kafka\data\kafka:/var/lib/kafka/data" `
  -e KAFKA_KRAFT_MODE=true `
  -e KAFKA_NODE_ID=1 `
  -e KAFKA_PROCESS_ROLES=broker,controller `
  -e KAFKA_LISTENERS="PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093" `
  -e KAFKA_LISTENER_SECURITY_PROTOCOL_MAP="PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT" `
  -e KAFKA_ADVERTISED_LISTENERS="PLAINTEXT://localhost:9092" `
  -e KAFKA_CONTROLLER_QUORUM_VOTERS="1@localhost:9093" `
  -e KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER `
  apache/kafka:4.1.1
```
#### 创建 Topic
```bash
docker exec kafka /opt/kafka/bin/kafka-topics.sh --create --topic test-topic --partitions 1 --replication-factor 1 --bootstrap-server localhost:9092
```

| 参数                                | 说明                           |
| --------------------------------- | ---------------------------- |
| –-create                          | 创建一个新 Topic                  |
| –-topic test-topic                | 给这个 Topic 起个名字叫 `test-topic` |
| –-partitions 1                    | 将这个 Topic 拆分成几块来存储           |
| –-replication-factor 1            | 这条消息要存几份（备份数）                |
| –-bootstrap-server localhost:9092 | 告诉脚本去哪里连接 Kafka 服务端          |

> [!翻译] “喂 Docker，帮我进到那个叫 `kafka` 的容器里，找到 `kafka-topics.sh` 这个工具，让它连接**本机的9092端口**，**创建**一个名字叫 `test-topic` 的主题，只需要 **1个分区**，数据只存 **1份**。”

### KRaft集群
#### 生成cluster-id
```bash
docker run --rm apache/kafka:4.1.1 /opt/kafka/bin/kafka-storage.sh random-uuid
```
它会生成一个ID例如：
```bash
WMHfX7bXRluXXotbmylxmQ
```
#### docker-compose.yml
```yml
version: '3.8'
services:
  kafka1:
    image: apache/kafka:4.1.1
    container_name: kafka1
    environment:
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_NODE_ID: 1
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:29093,2@kafka2:29093,3@kafka3:29093
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:29093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka1:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_LOG_DIR: /var/lib/kafka/data
      CLUSTER_ID: WMHfX7bXRluXXotbmylxmQ
    ports:
      - "19092:9092"
        
  kafka2:
    image: apache/kafka:4.1.1
    container_name: kafka2
    environment:
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_NODE_ID: 2
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:29093,2@kafka2:29093,3@kafka3:29093
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:29093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka2:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_LOG_DIR: /var/lib/kafka/data
      CLUSTER_ID: WMHfX7bXRluXXotbmylxmQ
    ports:
      - "19093:9092"

  kafka3:
    image: apache/kafka:4.1.1
    container_name: kafka3
    environment:
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_NODE_ID: 3
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:29093,2@kafka2:29093,3@kafka3:29093
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:29093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka3:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_LOG_DIR: /var/lib/kafka/data
      CLUSTER_ID: WMHfX7bXRluXXotbmylxmQ
    ports:
      - "19094:9092"
```
#### 启动集群
```bash
docker compose up -d
```
#### 测试集群
```bash
docker exec -it kafka1 bash -c "/opt/kafka/bin/kafka-topics.sh --create --topic test --bootstrap-server kafka1:9092 --partitions 3 --replication-factor 3"
```
