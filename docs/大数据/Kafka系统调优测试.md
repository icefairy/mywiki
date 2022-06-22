# kafka调优
> 准备工作，创建testtopic主题设置复制份数为1 分区数为节点数x2 delete.retention.ms=360000
> 2022-2-22 11:35:04
> DATA_IN: (100% coverage, -975 lag) 
> PIC_IN1: (100% coverage, 42926 lag) 
> PIC_IN2: (100% coverage, -181 lag) 
> PIC_IN3: (100% coverage, -32 lag) 

## spw调整前
vm.dirty_background_bytes = 0
vm.dirty_background_ratio = 10
vm.dirty_bytes = 0
vm.dirty_expire_centisecs = 3000
vm.dirty_ratio = 20
vm.dirty_writeback_centisecs = 500
vm.dirtytime_expire_seconds = 43200

## 测试方法：
./bin/kafka-producer-perf-test.sh --topic testtopic --print-metrics --record-size 102400 --throughput -1 --num-records 100000 --producer-props bootstrap.servers=ip1:9092,ip2:9092,ip3:9092,ip4:9092,ip5:9092

## 测试结果
29578 records sent, 5914.4 records/sec (577.58 MB/sec), 51.7 ms avg latency, 373.0 ms max latency.
35348 records sent, 7068.2 records/sec (690.25 MB/sec), 46.3 ms avg latency, 598.0 ms max latency.
34842 records sent, 6967.0 records/sec (680.37 MB/sec), 46.8 ms avg latency, 619.0 ms max latency.
100000 records sent, 6630.420369 records/sec (647.50 MB/sec), 48.31 ms avg latency, 619.00 ms max latency, 14 ms 50th, 217 ms 95th, 407 ms 99th, 559 ms 99.9th.

## spw调整后
vm.dirty_background_bytes = 0
vm.dirty_background_ratio = 20
vm.dirty_bytes = 0
vm.dirty_expire_centisecs = 3000
vm.dirty_ratio = 40
vm.dirty_writeback_centisecs = 12000
vm.dirtytime_expire_seconds = 43200
vm.max_map_count=262144
设置内存放到kafka启动用户的bash_profile中比如root的在/root/.bash_profile
export KAFKA_HEAP_OPTS=--Xms6g --Xmx6g
然后用source /root/.bash_profile生效，再启动节点
## 测试方法：
./bin/kafka-producer-perf-test.sh --topic writetest --print-metrics --record-size 102400 --throughput -1 --num-records 200000 --producer-props bootstrap.servers=ip1:9092,ip2:9092,ip3:9092,ip4:9092,ip5:9092

## 测试结果
39344 records sent, 7868.8 records/sec (768.44 MB/sec), 39.5 ms avg latency, 210.0 ms max latency.
44958 records sent, 8991.6 records/sec (878.09 MB/sec), 36.2 ms avg latency, 135.0 ms max latency.
100000 records sent, 8584.427848 records/sec (838.32 MB/sec), 37.29 ms avg latency, 210.00 ms max latency, 22 ms 50th, 134 ms 95th, 165 ms 99th, 197 ms 99.9th.
40639 records sent, 8127.8 records/sec (793.73 MB/sec), 40.0 ms avg latency, 7370.0 ms max latency.
46095 records sent, 9219.0 records/sec (900.29 MB/sec), 35.4 ms avg latency, 132.0 ms max latency.
49666 records sent, 9933.2 records/sec (970.04 MB/sec), 32.8 ms avg latency, 118.0 ms max latency.
48732 records sent, 9746.4 records/sec (951.80 MB/sec), 33.5 ms avg latency, 154.0 ms max latency.
46382 records sent, 9267.1 records/sec (904.99 MB/sec), 35.1 ms avg latency, 112.0 ms max latency.
46790 records sent, 9358.0 records/sec (913.87 MB/sec), 34.8 ms avg latency, 150.0 ms max latency.
300000 records sent, 1876.912104 records/sec (183.29 MB/sec), 35.64 ms avg latency, 60001.00 ms max latency, 20 ms 50th, 110 ms 95th, 131 ms 99th, 145 ms 99.9th.