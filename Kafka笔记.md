## dokcer启动kafka和zookeeper
```linux
docker run -d --name zookeeper -p 2181:2181 -v D:\kafka\data\zookeeper:data zookeeper:3.9
```
```linux
docker run -d --name kafka `
	-p 9092:9092 `
	-v D:\kafka\data\kafka:/var/lib/kafka/data `
	-e KAFKA_ZOOKEEPER_CONNECT=host.docker.internal:2181 `
	-e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 `
	-e KAFKA_BROKER_ID=1 `
	-e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 `
	--link zookeeper `
	apache/kafka:4.1.1
```
