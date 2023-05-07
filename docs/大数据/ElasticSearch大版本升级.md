# 大版本升级方案


## 6.x 到7.x

+ 如果版本不是6.8.23则先升级到6.8.23
+ 6.8.23再升级到7.x



## 滚动更新

```bash
# 关闭自动分片迁移
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "primaries"
  }
}

#进行一次写盘
POST _flush/synced
#停止所有机器学习任务
#停止一个节点进行升级
#用新的程序或者镜像启动节点（注意配置文件必要设置，例如发现地址、集群名称等）
#升级必要的插件
#开启自动分片迁移
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": null
  }
}
#查看集群状态变绿后再进行后续动作
GET _cat/health?v
#按照上面步骤升级其他节点
……
#最后通过下面网址查看所有节点的版本
GET /_cat/nodes?h=ip,name,version&v

```

```
#例子
docker run --name esdata -it --net host -e node.name=esdata -e xpack.security.enabled=false -e cluster.name=dev   -v /data/esdata:/usr/share/elasticsearch/data es624

docker run --name esdata6823 -it --net host -e node.name=esdata -e xpack.security.enabled=true -e xpack.security.transport.ssl.enabled=true -e cluster.name=dev   -v /data/esdata:/usr/share/elasticsearch/data elasticsearch:6.8.23

docker run --name esdata7177 -it --net host -v /home/ice/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml -e node.name=esdata -e xpack.security.enabled=true -e xpack.security.transport.ssl.enabled=true -e cluster.name=dev   -v /data/esdata:/usr/share/elasticsearch/data elasticsearch:7.17.7
```
