# Flink常用connector支持情况
|连接器名 | 流输入 | 流输出 | 批输入 | 批输出|
|----|----|---|---|---|
|ApacheKafka | 1 | 1 | 无界扫描 | 1|
|cassandra | 0 | 1|
|kinesis | 1 | 1 | 无界扫描 | 1|
|Elasticsearch | 0 | 1|
|FileSystem（streaming or batch） | 0 | 1 | 边界和无界扫描加检索 | 1|
|rabbitMQ | 1 | 1|
|ApacheNiFi | 1 | 1|
|TwitterStreamingAPI | 1 | 0|
|googlepubsub | 1 | 1|
|jdbc | 0 | 1 | 边界扫描加检索 | 1|
|ApacheActiveMQ | 1 | 1|
|ApacheFlume | 0 | 1|
|Redis | 0 | 1|
|Netty | 1 | 0|
|ApacheHBase |  | 1 | 边界扫描加检索 | 1|
|ApacheHive |  | 1 | 无界和边界扫描加检索 | 1|
|mongodb-cdc | 1 | 0 | 1 | 0|
|mysql-cdc | 1 | 0 | 1 | 0|
|oceanbase-cdc | 1 | 0 | 1 | 0|
|oracle-cdc | 1 | 0 | 1 | 0|
|postgres-cdc | 1 | 0 | 1 | 0|
|sqlserver-cdc | 1 | 0 | 1 | 0|
|tidb-cdc | 1 | 0 | 1 | 0|
|starrocks|||1|1