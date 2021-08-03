# Kafka常用命令

## 针对某个主题单独设置数据过期时间

> 可以通过kfkmgr进入相应集群选择对应的主题后点击 “update config”，进入后在retention.ms中设置相应的过期时间即可，单位是毫秒，并设置cleanup.policy为delete即可

## 设置消费坐标

```
#更新到当前group最初的offset位置
./bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group test-group --reset-offsets --all-topics --to-earliest --execute
#更新到指定的offset位置
./bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group test-group --reset-offsets --all-topics --to-offset 500000 --execute
#更新到当前offset位置（解决offset的异常）
./bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group test-group --reset-offsets --all-topics --to-current --execute
#offset位置按设置的值进行位移
./bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group test-group --reset-offsets --all-topics --shift-by -100000 --execute
#offset设置到指定时刻开始
./bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group test-group --reset-offsets --all-topics --to-datetime 2017-08-04T14:30:00.000
```