## 单机（swarm集群）部署

* 此方法部署的集群没有设置持久卷，可以使用nfs之类的方式设置统一的存储或者采用另外一种分布式部署的方式来进行

```bash
image="toughiq/mariadb-cluster"
rootpwd=`openssl rand -base64 16`
echo $rootpwd
/uDCWlDUdyVAPsFM8LqC7A==
docker network create mydbnet
#初始化集群
clustername=dbcluster
docker service create --name $clustername \
--network mydbnet \
--replicas=1 \
-e MYSQL_ROOT_PASSWORD=$rootpwd \
--env DB_SERVICE_NAME=$clustername \
toughiq/mariadb-cluster

#拉伸集群(数量最少3个，最好是奇数，如果设置为2的话其中一个掉线会导致集群无法使用）
docker service scale $clustername=3
```

## 分布式部署

* 不依赖swarm网络，直接使用主机网络
* 结合seaweedfs的docker存储插件参考["seaweedfs作为分布式存储插件"](siyuan://blocks/20210929161920-ova65mm)

```bash
#安装并配置好seaweedfs存储插件
#创建持久化存储卷
docker volume create -d seaweedfs mdb1

#初始化集群
image="toughiq/mariadb-cluster"
rootpwd=`openssl rand -base64 16`
echo $rootpwd
#hgvMgQRJGEyeYQ3oYWc07g==
nodename=mdb1
docker run -itd --name $nodename --hostname $nodename --restart=always -p 3306:3306 $image
```