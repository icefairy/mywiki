# 带用户密码的kafka搭建
> 2.x系列
## zookeeper
```bash
#更多参数参考：https://hub.docker.com/r/bitnami/zookeeper
mkdir /data/zkdata
chmod 777 -R /data/zkdata
docker run -itd --restart=always --name zookeeper -v /data/zkdata:/bitnami/zookeeper -e ZOO_SERVER_ID=1 -e ZOO_CLIENT_USER=zkuser -e ZOO_CLIENT_PASSWORD=zk_pass -e ZOO_ENABLE_AUTH=yes -p 2181:2181 bitnami/zookeeper:3.8
```

## kafka
```bash
mkdir /data/kfkdata
chmod 777 -R /data/kfkdata
docker run -itd --name kfk1 --restart=always -e KAFKA_CFG_ZOOKEEPER_CONNECT=192.168.240.79:2181  -e KAFKA_INTER_BROKER_USER=kfkuser -e KAFKA_INTER_BROKER_PASSWORD=kfkpwd -e KAFKA_HEAP_OPTS="-Xmx2048m -Xms1024m" -e KAFKA_ZOOKEEPER_PROTOCOL="SASL" -e KAFKA_ZOOKEEPER_USER=zkuser -e KAFKA_ZOOKEEPER_PASSWORD=zk_pass -e KAFKA_CLIENT_USERS=kfkuser,kfkconsumer,kfkproducer -e KAFKA_CFG_LISTENERS=SASL://0.0.0.0:9092 -e KAFKA_CLIENT_PASSWORDS=kfkpwd,kfkcon_pwd,kfkpro_pwd -e KAFKA_KRAFT_CLUSTER_ID=1 -v /data/kfkdata:/bitnami/kafka -p 9092:9092 -e ALLOW_PLAINTEXT_LISTENER=no bitnami/kafka:2.8.1
```