## 惯例

+ 只在ES中写入要搜索的字段。
+ 写入的数据总量尽量控制在filesystem cache设置的体积之内，能较好的保证搜索效率，如果超过了此体积需要对热数据做预加载。
+ 对数据进行冷热分类，尽可能将更高读写性能的设备用来存储热数据。
+ 对需要关联的数据建议在数据库中关联好再入到ES中。
+ 鉴于ES深度分页性能较差（体现在越往后翻页耗时越久），建议尽可能不给随意跳转到第N页的选项而是采用类似加载更多的机制，或者采用scrollapi的形式来进行翻页。

## 查看集群节点的磁盘状态

http://127.0.0.1:9200/_cat/allocation?v&pretty

## ES滚动方式获取更多数据

`select /*! USE_SCROLL(10,600000)*/id,person_img_url from person_pass where db_name='yh' and es_time between '2020-06-17 10:45:35' and '2020-06-24 00:00:00' order by es_time desc`

会返回一个scrollid,然后将id放到url中获取后续数据,一直到拿到的数据为空代表读取完了 ip:9200/_search/scroll?scroll=1m&scroll_id=DnF1ZXJ5VGhlbkZldGNoBQAAAAAAEwMtFlMxUng2Q0pwUkNxTV9yZ0plWktiLXcAAAAAABMDLhZTMVJ4NkNKcFJDcU1fcmdKZVpLYi13AAAAAAARXskWMklrckhac0ZSX3lGN0pfSnZjVWJjQQAAAAAAEV7KFjJJa3JIWnNGUl95RjdKX0p2Y1ViY0EAAAAAABJAKxZUeEMtTTU0elF4eXRTTVMteHJlWFRR

## 健康状态处理

### 检查状态

curl [http://ip:9200/_cluster/health?pretty](http://21.0.22.95:29200/_cluster/health?pretty)

```bash
{
  "cluster_name" : "docker-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 5,
  "number_of_data_nodes" : 5,
  "active_primary_shards" : 430,
  "active_shards" : 860,
  "relocating_shards" : 2,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

如果返回的数据中 status不是green就需要进行相应处理；

## 检查所有数据分片状态

curl [http://ip:9200/_cat/shards](http://21.0.22.95:29200/_cat/shards)

```
person_warn                 1 p STARTED          46 144.7kb 10.0.2.105 esdata2
person_warn                 1 r STARTED          46 144.7kb 10.0.2.108 esdata5
person_warn                 2 r UNASSIGNED          37 140.1kb 10.0.2.108 esdata5
person_warn                 2 p STARTED          37 140.1kb 10.0.2.107 esdata4
person_warn                 3 p STARTED          47 110.5kb 10.0.2.105 esdata2
person_warn                 3 r STARTED          47 110.5kb 10.0.2.107 esdata4
person_warn                 4 r STARTED          41 221.5kb 10.0.2.108 esdata5
person_warn                 4 p STARTED          41 221.5kb 10.0.2.107 esdata4
person_warn                 0 r STARTED          28   172kb 10.0.2.106 esdata3
person_warn                 0 p STARTED          28   172kb 10.0.2.105 esdata2

```

如果其中有UNASSIGNED状态则需要进行一下数据分片的重新分配（再平衡）

## 设置复制份数

```
curl -XPUT -H 'Content-Type: application/json' 'ip:9200/index/_settings' -d ' { "index" : { "number_of_replicas" : 0 } }'
```

## 解除只读

curl -XPUT -H 'Content-Type:application/json' http://ip:9200/_all/_settings -d '{"index.blocks.read_only_allow_delete":null}'

## 重新平衡数据分片

curl 'http://ip:9200/_cluster/reroute?retry_failed=true' -X POST

```bash
{
"acknowledged": true,
……
}
```

## 重置分片

post 192.168.240.14:9200/_cluster/reroute

```json
{
     "commands": [
        {
            "allocate_empty_primary": {
                "index": "newinfos",
                "shard": 1,
                "node": "esdata7",
              "accept_data_loss":true
          }
	}
    ]
  }
```

## 踢出节点

PUT _cluster/settings
{
  "transient": {
    "cluster.routing.allocation.exclude._ip":"10.10.2.254"
  }
}
注意此方法只能一能一个踢，踢完一个（数据文件夹占用的空间很小比如在1MB以内），停止这个节点的程序，再踢下一个

## 查询各节点的磁盘占用量

```
http://ip:9200/_cat/nodes?v&h=http,version,disk.total,disk.used,disk.used_percent
```

## es状态处理

查看集群状态
http://1.2.3.156:9200/_cluster/health?pretty
{
  "cluster_name" : "docker-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 10,
  "number_of_data_nodes" : 10,
  "active_primary_shards" : 486,
  "active_shards" : 962,
  "relocating_shards" : 20, #这里代表在平衡数据
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 2,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 695,
  "active_shards_percent_as_number" : 100.0
}

查看每个节点的磁盘占用情况，来分析是否数据不均衡（磁盘容量差不多的前提下）
http://1.2.3.156:9200/_cat/allocation?v

查看分片状态（包括正在迁移的分片）
http://1.2.3.156:9200/_cat/shards?v

查看节点状态（负载，内存占用等）
http://1.2.3.156:9200/_cat/nodes?v

查看索引数据量以及占用磁盘情况
http://1.2.3.156:9200/_cat/indices?v

查看挂起的任务
http://1.2.3.156:9200/_cat/pending_tasks?v

查看索引别名
http://1.2.3.156:9200/_cat/aliases?v

查看线程池
http://1.2.3.156:9200/_cat/thread_pool?v

查看bulk线程池（如果这里某个节点出现大量排队或者拒绝就需要做出处理了）
http://1.2.3.156:9200/_cat/thread_pool/bulk/?v



## 迁移数据到新的索引

```
POST localhost:9200/_reindex
{
  "source": {
    "index": "indexName",
    "type": "typeName",
    "query": {
      "term": {
        "name": "shao"
      }
    }
  },
  "dest": {
    "index": "newIndexName"
  }
}
```
