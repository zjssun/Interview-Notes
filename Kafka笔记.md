## dokcer启动kafka
```linux
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
