# 生产环境kafka配置说明

## 规划步骤
+ 根据需求计算数据量以及要保存的时间并预估所需的磁盘容量。
+ 为保证高可用性数据至少设置2个副本。
+ 对磁盘进行读写速度测试（关注顺序读写即可）
+ 如果不开压缩的话cpu核心数不需要太多8核就行。
+ 机器内存最低64G，建议128G起步。
+ 根据高峰时段的数据量计算所需的并发数，单台物理机可以支持4~5万并发，根据数据量以及副本数计算所需的机器数量。
+ 根据允许消息大小设置最大消息的尺寸。
+ 按照需求规划网络层面，建议直接万兆接入。
+ broker的分区数规划，分区数越多占用内存越多且选举leader耗时就越久。
+ 因为kafka使用页面缓存来作为重要的提升性能的手段，所以kafka不要和其他应用组件部署在同一个服务器。
+ 磁盘使用xfs格式进行加载，并设置参数noatime避免写入最后访问时间降低io性能。

## 服务器配置
+ 所有节点ntp校时设置。
+ 所有节点最大文件打开数量ulimit -n 65535并设置到/etc/security/limits.conf持久化存储。
+ bash_profile中添加如下配置并用source使其生效
```
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g -XX:MetaspaceSize=96m -XX:+UseG1GC -XX:MaxGCPauseMillis=20        -XX:InitiatingHeapOccupancyPercent=35 -XX:G1HeapRegionSize=16M        -XX:MinMetaspaceFreeRatio=50 -XX:MaxMetaspaceFreeRatio=80"
```
```
#sysctl.conf 设置好后通过sysctl -p生效
vm.swappiness=1
vm.max_map_count=262144
#socket mem config
net.core.wmem_default=131072
net.core.rmem_default=131072
net.core.wmem_max=20971520
net.core.rmem_max=20971520
#tcp mem config:min default max(<socket mem max)
net.ipv4.tcp_wmem=4096 65536 20480000
net.ipv4.tcp_rmem=4096 65536 20480000
net.ipv4.tcp_window_scaling=1
net.ipv4.tcp_max_syn_backlog=4096
net.core.somaxconn=4096
net.core.netdev_max_backlog=4000

```
## zookeeper配置（kafka ver<3.x的需要配置zookeeper）
+ zookeeper个数采用单数，3个或者5个，不建议超过5个否则因为一致性协议会导致降低整个zk集群的性能。
```bash
mkdir -p /data/zkdata
echo 1 > /data/zkdata/myid
```
```
#zoo.cfg
tickTime=2000
dataDir=/data/zkdata
clientPort=2181
#tickTime的倍数2000*20=40秒
initLimit=20
#tickTime的倍数2000*5=10秒
syncLimit=5
server.1=ipaddress1:2888:3888
server.2=ipaddress2:2888:3888
server.3=ipaddress3:2888:3888
```


## kafka broker配置

```bash
mkdir -p /data/kfkdata

```
```
#server.properties
#正整数，每个节点不同
broker.id=1
zookeeper.connect=zk1:2181/cluster1,zk2:2181/cluster1,zk3:2181/cluster1
log.dirs=/data/kfkdata
#启动读取以及关闭写入的线程数，如果磁盘在raid上可以适当增加点
num.recovery.threads.per.data.dir=1
#允许自动创建主题
auto.create.topics.enable=true
#自动创建的主题的分区数，根据所需的每秒吞吐量进行估算
num.partitions=20
#默认副本数
default.replication.factor=2
#数据保留时间 默认168小时
log.retention.hours=168
#设置单个分区最大可以保留多少体积的数据（1GB）,上述时间和本设置的体积两者是或的关系，有一个满足就会进行清理
log.retention.bytes=1000000000
#单个日志文件的最大尺寸，默认1GB，当文件满1GB后才会关闭句柄进入可以被上述两种规则清理的范围，如果单条数据很小且日增数据量也很小，此值需要相应的改小，否则会很久都无法自动清理。
log.segment.bytes=1000000000
#单个日志的关闭时间，无默认值，时间或者体积任意一个达到条件会关闭此日志文件。
log.segment.ms=86400000
#最大消息体积默认1MB，如果要处理的数据单条超过此值就无法进来，需要根据最大消息尺寸来规划设置
message.max.bytes=10000000
#集群间复制消息的大小限制
replica.fetch.max.bytes=10000000
#消费者消费消息大小限制为fetch.message.max.bytes如果小于服务器设置name将无法下载到较大的数据
vm.dirty_ratio = 60
```

## kafka consumer配置
```
#每个分区最大返回10MB，如果消费者分到了10个分区将占用100MB来解析数据
max.partition.fetch.bytes=10485760
auto.offset.reset=earliest
#自动提交由下面两个配置决定
enable.auto.commit=true
auto.commit.interval.ms=1000
client.id=clientid
max.poll.records=500
```