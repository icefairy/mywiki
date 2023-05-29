# kafka性能测试



 Kafka 压力测试文档

# 1    概述

## 1.1  **测试背景**

在云平台研发SR IAD的过程中，出现SR IAD对硬件资源消耗较为严重的情况，其中在云平台研发中利用Kafka软件对流式数据进行数据处理。我们针对Kafka高吞吐量的特性，对kafka进行压力测试。

## 1.2  **测试目标**

测试kafka的吞吐性能(Producer/Consumer性能)。我们主要对分区、磁盘和线程等影响因素进行测试。

# 2    测试条件

6台配置相同的服务器搭建的两套相同的集群环境

**2.1****测试环境**

**硬件条件**

| 序号 | 硬件  | 配置                                                                                                                                          | 备注         |
| ---- | ----- | --------------------------------------------------------------------------------------------------------------------------------------------- | ------------ |
| 1    | CPU   | E5-2640v2 *2                                                                                                                                  |              |
| 2    | 内存  | 128G                                                                                                                                          |              |
| 3    | 硬盘  | 共挂载15块磁盘，剩余空间10T左右                                                                                                               |              |
| 4    | 网络  | Intel I350 Gigabit Network Connection *4Intel 82599ES 10-Gigabit SFI/SFP+ Network Connection *2Intel I350 Gigabit Fiber Network Connection *2 | 使用万兆网卡 |
| 5    | Kafka | 3台单磁盘服务组成的kafka集群                                                                                                                  |              |

**软件版本：**

| 序号 | 软件           | 版本          | 备注 |
| ---- | -------------- | ------------- | ---- |
| 1    | CentOS         | 7.3           |      |
| 2    | Hadoop         | 2.7.3         |      |
| 3    | HBase          | 1.1.7         |      |
| 4    | Spark          | 1.6.2         |      |
| 5    | Elastic Search | 2.4.1         |      |
| 6    | Scala          | 2.11.8        |      |
| 7    | Kafka          | 2.11-0.10.0.1 |      |
| 8    | Flume          | 1.7           |      |

**系统架构：**

**Kafka默认配置：（CDH里的配置）**

（1） broker中默认配置

1）网络和io操作线程配置

 \# broker处理消息的最大线程数 （一般不需要修改）

num.network.threads=3

\# broker处理磁盘IO的线程数

num.io.threads=8

2）log数据文件刷盘策略

   为了大幅度提高producer写入吞吐量，需要定期批量写文件

   \#每当producer写入10000条消息时，刷数据到磁盘

log.flush.interval.messages=10000

\# 每间隔1秒钟时间，刷数据到磁盘

log.flush.interval.ms=1000

3）日志保留策略配置

\# 保留三天，也可以更短

log.retention.hours=72

\# 段文件配置1GB，有利于快速回收磁盘空间，重启kafka加载也会加快

log.segment.bytes=1073741824

4）Replica相关配置

replica.lag.time.max.ms:10000

replica.lag.max.messages:4000

num.replica.fetchers:1

\#用来控制Fetch线程的数量。

default.replication.factor:1

\#自动创建topic时默认的Replica数量，一般建议在2~3为宜。

5）其他

num.partitions:1

\#分区数量

queued.max.requests:500

\#用于缓存网络请求的队列的最大容量

compression.codec:none

\#Message落地时是否采用以及采用何种压缩算法。

in.insync.replicas:1

\#指定每次Producer写操作至少要保证有多少个在ISR的Replica确认，一般配合request.required.acks使用。过高可能会大幅降低吞吐量。

（2） producer 配置

buffer.memory:33554432 (32m)

\#在Producer端用来存放尚未发送出去的Message的缓冲区大小。

compression.type:none

\#生产者生成的所有数据压缩格式，默认发送不进行压缩。若启用压缩，可以大幅度的减缓网络压力和Broker的存储压力，但会对cpu 造成压力。

linger.ms:0

\#Producer默认会把两次发送时间间隔内收集到的所有Requests进行一次聚合然后再发送，以此提高吞吐量，这个参数为每次发送增加一些delay，以此来聚合更多的Message。

batch.size:16384

\#Producer会尝试去把发往同一个Partition的多个Requests进行合并，batch.size指明了一次Batch合并后Requests总大小的上限。如果这个值设置的太小，可能会导致所有的Request都不进行Batch。

acks:1

\#设定发送消息后是否需要Broker端返回确认。

0: 不需要进行确认，速度最快。存在丢失数据的风险。

1: 仅需要Leader进行确认，不需要ISR进行确认。是一种效率和安全折中的方式。

all: 需要ISR中所有的Replica给予接收确认，速度最慢，安全性最高，但是由于ISR可能会缩小到仅包含一个Replica，设置参数为all并不能一定避免数据丢失。

（3） Consumer

num.consumer.fetchers:1

\#启动Consumer的个数，适当增加可以提高并发度。

fetch.min.bytes:1

\#每次Fetch Request至少要获取多少字节的数据才可以返回。

\#在Fetch Request获取的数据至少达到fetch.min.bytes之前，允许等待的最大时长。

fetch.wait.max.ms:100

（4） 其他

zookeeper.session.timeout.ms=6s

message.max.bytes=1000000B

replica.fetch.max.bytes=1MB

num.producers=1

\### Number of Producers

num.streams=1

\###Number of Consumer Threads

producer.request.timeout.ms=30s

consumer.request.timeout.ms=40s

session.timeout.ms=30s

kafka.log4j.dir=/var/log/kafka

\##kafka数据的存放地址

**2.2** **测试方法**

压力发起：kafka官方提供的自测脚本

4项指标：总共发送消息量（以MB为单位），每秒发送消息量（MB/second），发送消息总数，每秒发送消息数（records/second）

监控信息：自定义脚本来记录服务情况，包括CPU、内存、磁盘、网络等情况。

（1） 生产者测试脚本kafka-producer-perf-test.sh

参数说明：

--topic topic名称，

--num-records 总共需要发送的消息数，

--record-size 每个记录的字节数，

--throughput 每秒钟发送的记录数，

--producer-props bootstrap.servers=localhost:9092 发送端的配置信息

（2）消费者测试脚本kafka-consumer-perf-test.sh

参数说明：

--zookeeper 指定zookeeper的链接信息，

   --topic 指定topic的名称，

--fetch-size 指定每次fetch的数据的大小，

--messages 总共要消费的消息个数，

**2.2.1 producer** **吞吐量测试方法**

1、基于原有配置下对kafka进行数量级的加压，选用一台或三台客户机先测试在没有消费者时的吞吐量。测试影响因素主要包括partitions、磁盘和thread三个主要因素，通过记录每秒钟条数（Records/s）、每秒日志量大小（MB/s）衡量吞吐量，并找到各个因素之间的规律。

注：

（1） 线程数的设置根据cpu内核数设置，本测试环境为16核（thread <=16）。

（2） Broker 基于物理环境设置（broker<=3），三个

2、通过修改以上3个影响因素测试kafka吞吐量。

3、测试命令

（1）创建topic

1)./kafka-topics.sh --zookeepr IP:2181 --create --topic test --partitions 3 --replication-factor 1

2)./ppt.sh --topic pc918 --num-records 50000000 --record-size 1000 --throughput 10000000 --threads 3 --producer-props

 bootstrap.servers=192.168.17.57:9092,192.168.17.64:9092,192.168.17.65:9092

注：ppt.sh 基于kafka-producer-perf-test.sh修改，增加了生产者的thread 选项。

**2.2.2** **consumer** **吞吐量测试方法**

1、基于原有配置下进行消费，并测试主要影响因素partitions、thread和磁盘等因素，选用一台或三台客户机先测试没有生产者时的吞吐量。通过记录Records/second、MB/second衡量吞吐量，并找到各个因素之间的规律。

2、通过修改影响因素测试kafka吞吐量。

3、测试命令

./kafka-consumer-perf-test.sh --zookeeper IP:2181 --topic test1 --fetch-size 1048576 --messages 10000000 --threads 3

**2.2.3** **消费与吞吐量**

  分别在一台或三台客户机同时开启生产者和消费者测试命令,进行测试

# 3    测试数据（部分样例）

3**.1** **写入测试(only生产者)**

### **3.1.1**   **Partition VS. Throughput**

    实验条件:3个Broker,1个Topic, Replication-factor=1，3个磁盘，throughput=1000w,num-records=2000w ，batch-size=16K ,1个客户端，1个producer

   测试项目：分别测试1，3，6，12，15，20，30，50个partition时的吞吐量

    表1 partition影响因子

| 类型      | partition | Records/second(avg) | MB/second(avg) | 延迟时间（avg） | 延迟时间(max) |
| --------- | --------- | ------------------- | -------------- | --------------- | ------------- |
| partition | 1         | 102760              | 98             | 297.65          | 929           |
| 3         | 283639    | 270.5               | 107.1          | 1965            |               |
| 6         | 332457    | 317.06              | 61.22          | 1085            |               |
| 9         | 305838    | 291.67              | 39.37          | 2842            |               |
| 12        | 344245    | 328.3               | 21.68          | 645             |               |
| 15        | 349846    | 333.64              | 20.51          | 618             |               |
| 20        | 349852    | 333.6               | 15.06          | 371             |               |
| 30        | 349870    | 333.66              | 15.47          | 843             |               |
| 50        | 346296    | 330.25              | 13.07          | 619             |               |

当partition的个数为3时，吞吐量成线性增长，当partition多于broker的个数时增长并不明显，甚至还有所下降。同时随着，partition个数的增多，延迟时间逐渐减少，当partition个数在3-6之间延迟时间降低较快，越大延迟时间降低趋于平稳。

### **3.1.2**   **Replica  VS. Throughput**

实验条件:3个Broker,1个Topic,3个磁盘，throughput=35w,num-records=5000w ，batch-size=16K ,1个客户端，1个producer

   测试项目：分别测试replication-factor为1，2，3，6时的吞吐量

    表2 Repica影响因子

| 类型        | replications | Records/second(avg)                                                                          | MB/second(avg) | 延迟时间（avg） | 延迟时间(max) |
| ----------- | ------------ | -------------------------------------------------------------------------------------------- | -------------- | --------------- | ------------- |
| replication | 1            | 349870                                                                                       | 333.66         | 12.69           | 713           |
| 2           | 349846       | 333.64                                                                                       | 10.8           | 392             |               |
| 3           | 345077       | 329.09                                                                                       | 36.36          | 897             |               |
| 6           | 错误         | Error while executing topic command : replication factor: 6 larger than available brokers: 3 |                |                 |               |

由表可知，replication-factor的个数应该不大于broker的个数，否则就会报错。随着replication-factor 个数的增加吞吐量减小，但并非是线性下降，因为多个Follower的数据复制是并行进行的，而非串行进行，因此基于数据的安全性及延迟性考虑，最多选择2-3个副本。

### **3.1.3**   **Thread VS. Throughput**

实验条件:3个Broker,1个Topic,3个磁盘，3个partitions , throughput=1000w,num-records=1000w ，batch-size=16K ,1个客户端，1个producer

测试项目：分别测试thread为1，2，3，4，5时的吞吐量

    表3 Thread 影响因子

| 类型   | partition | thread | Records/second(avg) | MB/second(avg) | 延迟时间（avg） | 延迟时间(max) |
| ------ | --------- | ------ | ------------------- | -------------- | --------------- | ------------- |
| thread | 3         | 1      | 265505              | 253.21         | 113.79          | 571           |
| 2      | 559252    | 533.35 | 93.75               | 388            |                 |               |
| 3      | 730300    | 696.47 | 68.96               | 337            |                 |               |
| 4      | 722700    | 689.22 | 42.49               | 343            |                 |               |
| 5      | 740137    | 705.85 | 36.88               | 283            |                 |               |

由表可知随着线程数的增加吞吐量不断增加，当线程数小于分区数时增长较快，大于分区数时增长较慢，并趋于平稳。目前单个客户端可以达700MB-800MB/s以上的网速流量。

### **3.1.4**   **磁盘** **VS. Throughput**

 实验条件:3个Broker,1个Topic，throughput=1000w,num-records=1000w ，batch-size=16K ,1个客户端，1个producer

 测试项目：分别测试磁盘个数为3，6，9时的吞吐量


表4 磁盘影响因子

| 类型 | partition | 磁盘个数 | Records/second(avg) | MB/second(avg) | 延迟时间（avg） | 延迟时间(max) |
| ---- | --------- | -------- | ------------------- | -------------- | --------------- | ------------- |
| 磁盘 | 3         | 3        | 265505              | 253.21         | 113.79          | 571           |
| 6    | 6         | 354886   | 338.45              | 58.13          | 512             |               |
| 9    | 9         | 363240   | 346.41              | 6.33           | 409             |               |

由表可知，磁盘个数的增加与partition的增加具有相关性，并非越多越好，partition的增加受broker的影响，因此磁盘的个数设置应在broker个数的1-3倍之间较为合适，同时随着磁盘个数的增加，平均延迟时间在逐渐减小，因此磁盘的数量会影响平均延迟时间。

### **3.1.5**   **Others VS. Throughput**

**1****、Network线程数**

Network线程数即broker处理消息的最大线程数 （一般不需要修改）。

实验条件:3个Broker,1个Topic，throughput=1000w,num-records=5000w ，batch-size=16K ,1个客户端，1个producer，9个磁盘，9个partitions

测试项目：分别测试线程个数为3，8时的吞吐量

表5 Network网络线程数

| net.io.thread | 生产者y99_thread | Records/second(avg) | MB/second(avg) | CPU(max%) |
| ------------- | ---------------- | ------------------- | -------------- | --------- |
| 3             | 3                | 516913              | 492.97         | 560.5     |
| 6             | 528966           | 504                 | 943.2          |           |
| 9             | 441789           | 421.32              | 799.3          |           |
| 8             | 3                | 629945              | 600.76         | 636       |
| 6             | 554865           | 529.16              | 855            |           |
| 9             | 543371           | 518.2               | 859            |           |

由表所示，增加网络线程数可以提高吞吐量，但随着生产者线程数增加逐渐趋于平稳，此时CPU最大值为998.2%，对于16核的CPU约占到10核。

**2****、测试客户机的影响**

由于在单个客户机测试时，网卡硬件条件的限制会影响测试的吞吐量，因此换用3台客户机测试，吞吐量的测试结果即为三个客户机吞吐总量。分别测试6个磁盘和9个磁盘在不同分区和线程数的情况，测试结果如表6所示。

    表6 客户机影响因素

| topic_partition_thread | 66             | 67                  | 68             | 总量                |                |                     |                |         |
| ---------------------- | -------------- | ------------------- | -------------- | ------------------- | -------------- | ------------------- | -------------- | ------- |
| Records/second(avg)    | MB/second(avg) | Records/second(avg) | MB/second(avg) | Records/second(avg) | MB/second(avg) | Records/second(avg) | MB/second(avg) |         |
| p66(3)                 | 439382         | 419.03              | 461382         | 440.01              | 449458         | 428.64              | 1350222        | 1287.68 |
| p66(6)                 | 471635         | 449.79              | 468059         | 446.38              | 466409         | 444.8               | 1406103        | 1340.97 |
| p66(12)                | 145292         | 138.56              | 145266         | 138.54              | 144017         | 137.35              | 434575         | 414.45  |
| p612(3)                | 498037         | 474.97              | 488181         | 465.57              | 584098         | 557.04              | 1570316        | 1497.58 |
| p612(6)                | 305658         | 291.5               | 305255         | 291.11              | 305715         | 291.55              | 916628         | 874.16  |
| p612(12)               | 102037         | 97.31               | 102032         | 97.31               | 101951         | 97.23               | 306020         | 291.85  |
| p99(3)                 | 473771         | 451.82              | 481621         | 459.31              | 498658         | 475.56              | 1454050        | 1386.69 |
| p99(6)                 | 207400         | 197.79              | 207337         | 197.73              | 206322         | 196.76              | 621059         | 592.28  |
| p99(12)                | 174175         | 166.11              | 178429         | 170.16              | 177564         | 169.34              | 530168         | 505.61  |

  由表6可以看出，在三台测试机下，同时向一个topic产生数据，生产者的数据总量是在增大的。在网络稳定的情况下，曾测试3个磁盘3个partition最大吞吐量在18000条左右 ，大小为1683.31MB/s，如图1所示。

其中在单个broker上cpu最大上线为700%-800%，内存利用率为10%-20%。在单台测试机情况下，曾测试单台测试下最大吞吐为80w条左右，大小约为80MB/s，由于资源限制三台机器总量可能达不到一台机器3倍的量。


    图1 3台机器的吞吐量

**3****、IO线程数**

IO线程数即broker处理磁盘IO的线程数

实验条件:3个Broker,1个Topic，throughput=1000w,num-records=5000w ，batch-size=16K ,3个客户端，1个producer，6个磁盘，6个partitions

测试项目：分别测试线程个数为8，12时的吞吐量

图2  线程数为8

    图3 线程数为12

  由图2，3可知，当线程数增加后，吞吐量有所增加，但增加比较缓和。

**4、生产者的个数**

  实验条件:3个Broker,1个Topic，throughput=1000w,num-records=5000w ，batch-size=16K ,3个客户端，1个producer，9个磁盘，9个partitions

测试项目：分别测试生产者个数为1，2，3时的吞吐量

 表7 不同生产者吞吐量

| thread | 1      | 2      | 3      |        |        |        |        |        |        |        |        |        |
| ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ |
| 1窗口  | 1窗口  | 2窗口  | 1窗口  | 2窗口  | 3窗口  |        |        |        |        |        |        |        |
| 条     | MB     | 条     | MB     | 条     | MB     | 条     | MB/s   | 条     | MB/s   | 条     | MB/s   |        |
| 3      | 629945 | 600.76 | 338900 | 323.2  | 328357 | 313.15 | 209465 | 199.76 | 205923 | 196.38 | 209129 | 199.44 |
| 6      | 554865 | 529.16 | 354172 | 337.77 | 382564 | 364.84 | 235881 | 224.95 | 236085 | 225.15 | 245225 | 233.87 |
| 9      | 543371 | 518.2  | 370090 | 352.95 | 359056 | 342.42 | 209646 | 199.93 | 221790 | 211.52 | 209058 | 199.37 |

   由表7 可知，在同一客户端下，开启不同的生产者，2个生产者和3个生产的总吞吐量基本等于1个生产者的总量。

 **5、压缩因素**

实验条件:3个Broker,1个Topic，throughput=1000w,num-records=5000w ，batch-size=16K ,1个客户端，1个producer，9个磁盘，9个partitions

测试项目：分别测试不同压缩类型的吞吐量，kafka支持的压缩类型包括none、Snappy、gzip、LZ4。

测试命令：

./ppt.sh --topic py --num-records 50000000 --record-size 1000 --throughput 10000000 --threads 3 --producer-props bootstrap.servers=192.168.17.68:9092,192.168.17.69:9092,192.168.17.70:9092 compression.type=gzip

通过运行时初始化参数看到修改值生效，如图所示。

    图4 初始化参数


表8 压缩因素

| 压缩类型 | Records/second(avg) | MB/second(avg) | 延迟时间（avg） | 延迟时间(max) | 耗时   | cpu max | cpu avg |
| -------- | ------------------- | -------------- | --------------- | ------------- | ------ | ------- | ------- |
| none     | 636472              | 606            | 122.56          | 3281          | 78s    | 899     | 545     |
| Snappy   | 392233              | 374.06         | 50.88           | 672           | 80s    | 670     | 574     |
| gzip     | 92653               | 88.36          | 2.38            | 386           | 8分59s | 959.5   | 522     |
| LZ4      | 618444              | 589.79         | 4               | 329           | 81s    | 887.7   | 480     |

由表可知snappy、gzip压缩可以大幅度的减缓网络压力和Broker的存储压力，同时降低平均延迟时间，LZ4压缩在降低平均延迟时间时最显著，gzip在生产数据时耗时最大。

## 3.2 **读取测试(only****消费者)**

### **3.2.1**   **Thread VS. Throughput**

实验条件:3个Broker,1个Topic,3个磁盘，throughput=1000w,num-records=5000w ，1个客户端

 测试项目：分别测试thread为1，2，3，6时的吞吐量

    表9  线程因素

| console&thread | Records/second(avg) | MB/second(avg) |
| -------------- | ------------------- | -------------- |
| p33(11)        | 340124              | 324            |
| p33(12)        | 799073              | 762            |
| p33(13)        | 973551              | 928            |
| p33(16)        | 963081              | 918            |

  由表可知，增加线程数会增加消费者的吞吐量，当小于3个线程时吞吐量增速较快，大于3个的时候趋于大体平稳，单个客户端消费者的吞吐量可达到928MB/s。同时，消费者吞吐量大于生产者的吞吐量并且处理消息的时间总大于生产者。因此在合理的配置下可以保证消息被及时处理。

### **3.2.2** **消费者** **VS. Throughput**

 实验条件:3个Broker,1个Topic,3个磁盘，throughput=1000w,num-records=5000w ，3个客户端，即三个消费者

  测试项目：分别测试消费者为1， 3时的吞吐量

  当消费者个数有1增为3时，吞吐量由324MB/s增加到1324MB/s。Consumer消费消息时以Partition为分配单位，当只有1个Consumer时，该Consumer需要同时从3个Partition拉取消息，该Consumer所在机器的I/O成为整个消费过程的瓶颈，而当Consumer个数增加至3个时，多个Consumer同时从集群拉取消息，充分利用了集群的吞吐率。对于同一个组内的消费者只允许一个消费者消费一个partition，不同组之间可以消费相同的partition。

### **3.2.3** **磁盘** **VS. Throughput**

 实验条件:3个Broker,1个Topic, throughput=1000w,num-records=5000w ，1个客户端

 测试项目：分别测试消费者为3，6，9时的吞吐量

    表10 磁盘影响因素

| console&thread | Records/second(avg) | MB/second(avg) |
| -------------- | ------------------- | -------------- |
| p36(33)        | 938827              | 895            |
| p66(33)        | 888458              | 847            |
| p96(33)        | 259475              | 247            |

由表看出，磁盘数增加并不能提高吞吐率，反而吞吐率相应下降。因此，在一定broker和partition 下可以按两者值合理增加磁盘。

**Consumer** **小结**

（1） Consumer 吞吐率远大于生产者，因此可以保证在合理的配置下，消息可被及时处理。

（2） Consumer 处理消息时所消耗单个broker的CPU一般在70%以下，所占内存也较小，属于轻量级进程。

（3） 本实验在三台客户端测得Consumer的最大吞吐量在2617MB/s

## 3.3 生产者&消费者

模拟类似真实的环境，生产消费过程

实验条件:3个Broker,1个Topic, throughput=1000w,num-records=5000w ，3个客户端，即三个消费者

   测试项目：分别测试磁盘为3，6，9时的吞吐量

（1） Producer

    表11 生产者吞吐量

| topic     | Records/second(avg) | MB/second(avg) |
| --------- | ------------------- | -------------- |
| pc36(366) | 1023426             | 976.02         |
| p66(366)  | 1319431             | 1258.31        |
| p612(366) | 1273905             | 1214.88        |
| p99(366)  | 989373              | 943.54         |
| p918(366) | 1159443             | 1105.73        |

（2） Consumer

    表12 消费者吞吐量

| topic     | Records/second(avg) | MB/second(avg) |
| --------- | ------------------- | -------------- |
| pc36(366) | 934007              | 889            |
| p66(366)  | 2259154             | 2153           |
| p612(366) | 1982417             | 1889           |
| p99(366)  | 649876              | 618            |
| p918(366) | 1313163             | 1817           |

由producer与consumer两表，测得生产的最大吞吐量为1258.31MB/s,消费最大吞吐量为2153MB/s。由此看出，与单生产、单消费时的吞吐量大体相当,所用CPU及内存量并未显著上升。
