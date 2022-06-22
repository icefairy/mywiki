# Citus集群使用

## 安装
```bash
curl https://install.citusdata.com/community/deb.sh | sudo bash
apt update 
apt-get -y install postgresql-14-citus-10.2
pg_conftool 14 main set shared_preload_libraries citus
sudo service postgresql restart
# and make it start automatically when computer does
sudo update-rc.d postgresql enable
sudo -i -u postgres psql -c "CREATE EXTENSION citus;"

```

## 使用
```sql
-- 修改连接数（重启生效）
alter system set max_connections = 1000;
--提前设置好pg_hba.conf允许指定主机通过某ip或者内部网络信任链接
--设置主节点连接信息
SELECT citus_set_coordinator_host('coord.example.com', 5432);
--添加worker节点
SELECT * from citus_add_node('worker-101', 5432);
SELECT * from citus_add_node('worker-102', 5432);
--#查看节点数
SELECT * FROM citus_get_active_worker_nodes();
--普通的创建表
create table icetest(id serial4,uname varchar(32));
--#转换为分区表
select create_distributed_table('icetest','id');
--#增加或者减少节点或者重新平衡表数据
SELECT rebalance_table_shards('companies');
--广播字典表
create table mydict(code varchar(32),name varchar(255));
select create_reference_table('mydict');
--把分布式表变成普通表
SELECT undistribute_table('mydict');
--在所有worker执行
SELECT run_command_on_workers($cmd$ SHOW work_mem; $cmd$);
--创建分布式udf
create_distributed_function
--设置复制份数保持节点高可用（确保有足够worker的情况下）
SET citus.shard_replication_factor = 2;
--使用列式存储（一般用于冷数据）
CREATE TABLE contestant (
    handle TEXT,
    birthdate DATE,
    rating INT,
    percentile FLOAT,
    country CHAR(3),
    achievements TEXT[]
) USING columnar;
--转换表为行存储
SELECT alter_table_set_access_method('contestant', 'heap');
--转换表为列存储
SELECT alter_table_set_access_method('contestant', 'columnar');
--更新节点信息
select citus_update_node(node_id,node_name,node_port);
--禁用/激活节点
citus_disable_node(name,port)
citus_activate_node(name,port)
citus_remove_node(name,port)
--添加备用主节点
citus_add_secondary_node(nodname,nodeport,primaryname,primaryport,nodeclustername)
```
## 集群模式配置
```sql
--初始化设置
alter system set citus.shard_count=32; --分片数默认32 建议配置为CPU总核数的2~4倍；
alter system set citus.shard_replication_factor=1; --副本数
-- 第一种模式 高并发写入模式   默认模式
-- 注意：支持多节点写入（高并发写入，可以直接在数据节点写入，避免master负载过高，目前发现的约束为自增序列必须用serial8这个级别，每个数据节点会有自己的分段，比如A从1~10000  B从 20000~100000），可以考虑同步时间的前提下用数据库入库时间来做增量同步或者用cdc模式来增量同步到es之类
-- 这种模式不直接支持复制份数的设置，有节点挂了将会影响数据查询（可以临时禁用挂掉的节点，使得查询可用，但掉线节点的数据暂时不存在，修好后再激活节点）

--第二种 citus自带的高可用模式
-- alter system set citus.replication_model='statement'; --这是全局设置（但不对当前session生效） 这个设置在最新版中已经启用了会有相应提示，让直接设置下面这个，再创建的新表就会是有复制份数的啦
alter system set citus.shard_replication_factor=2;
```

## 分区分布式表与自动维护分区
```sql
--创建分区表
CREATE TABLE github_events (
  event_id bigint,
  event_type text,
  event_public boolean,
  repo_id bigint,
  payload jsonb,
  repo jsonb,
  actor jsonb,
  org jsonb,
  created_at timestamp
) PARTITION BY RANGE (created_at);
--分布式分发
SELECT create_distributed_table('github_events', 'repo_id');
--自动建分区
SELECT cron.schedule('create-partitions', '0 0 1 * *', $$
  SELECT create_time_partitions(
      table_name         := 'github_events',
      partition_interval := '1 month',
      end_at             := now() + '12 months'
  )
$$);
--可选的：自动清理过期的不再被需要（或者已经转入冷数据表归档的）数据
SELECT cron.schedule('drop-partitions', '0 0 1 * *', $$
  CALL drop_old_time_partitions(
      'github_events',
      now() - interval '12 months' /* older_than */
  );
$$);
```

## 归档冷数据案例
```sql
CREATE TABLE github_columnar_events  (
  event_id bigint,
  event_type text,
  event_public boolean,
  repo_id bigint,
  payload jsonb,
  repo jsonb,
  actor jsonb,
  org jsonb,
  created_at timestamp
) PARTITION BY RANGE (created_at);

SELECT create_time_partitions(
  table_name         := 'github_columnar_events',
  partition_interval := '2 hours',
  start_from         := '2015-01-01 00:00:00',
  end_at             := '2015-01-01 08:00:00'
);
--设置旧分区为列存压缩模式
CALL alter_old_partitions_set_access_method(
  'github_columnar_events', now() - interval '6 months',
  'columnar'
);
```

## 线上迁移
+ 提前设置好pg_hba.conf允许指定主机通过某ip或者内部网络信任链接
+ 复制结构
```bash
pg_dump --schema-only
```
+ 主节点开启逻辑复制（重启生效）
```
wal_level = logical
max_replication_slots = 5 # has to be > 0
max_wal_senders = 5       # has to be > 0
```
```sql
--创建用户或者直接使用postgres用户
CREATE ROLE fzuser WITH REPLICATION PASSWORD 'Fz123456' LOGIN;
```
+ 从节点
```

```
+ 开始复制
+ 确定数据库已经同步后，可以结合pgbouncer来快速切换所有客户端到新库


## jdbc写入测试
--往主节点写
2 副本
10字段 10w条100秒=1000/s

1副本
10字段 10w条41秒=2400+/s

单机
10字段 10w条2.8秒=35000+/s
10字段 50w数据12秒=41666/s
--主节点建表然后直接往数据节点写
2 副本
10字段 10w条53秒=1886/s

1副本
10字段 10w条30秒=3333+/s

-- 库内inser into
2副本
10字段20w数据2.3秒=86900+/s

1副本
10字段20w数据1秒=200000/s