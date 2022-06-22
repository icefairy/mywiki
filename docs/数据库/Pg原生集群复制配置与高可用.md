# PG原生集群复制配置与高可用

## 复制模式简介
### 流复制
> 基于文件系统的字节流形式的复制，特点为：主节点可以有多个从节点、主节点可以挂载从节点、主节点个数不限层级不限、主从可以同步/异步复制也可以混合存在。

> 无额外依赖。

### 逻辑复制
> 逻辑复制会在流复制的基础上将数据变动抽象为insert、delete等操作，经过发布节点sender进程向外发布，订阅节点拿到后进行操作重放，特点为：所有节点对等都可以读写、所有节点可以同时为发布或者订阅或者混合存在。

> 依赖wal2json插件


## 异步流复制
+ 主库配置pg_hba.conf允许备库的复制链接
```
host    replication     all             192.168.240.14/32       trust
```
+ 主库配置postgresql.conf
```
listen_address='*'
max_wal_senders=5 # 大于备库数量
wal_level=hot_standby
```
+ 初始化备库
```bash
pg_basebackup -h mdw -U postgres -F p -P -R -D /data/oscg/pgpool/data1 -p 11002 -l backuplog 
```
+ 备库的postgresql.conf添加主库链接信息
```
primary_conninfo='application_name=mdw1 user=postgres host=mdw port=11002 sslmode=disable sslcompression=1 target_session_attrs=any'
```
+ 生产备库标志文件
```bash
touch /data/oscg/pgpool/data1/standby.signal

```
+ 启动备库
```bash
pg_ctl start -D data1
```

+ 主节点查看状态
```sql
select * from pg_stat_replication;
```
## 逻辑复制
+ 主库配置pg_hba.conf允许备库的复制链接
```
host    replication     all             192.168.240.14/32       trust
```
+ 主库配置postgresql.conf
```
listen_address='*'
wal_level=logical
# 更改solts最大数量（默认值为10），flink-cdc默认一张表占用一个slots 下面开始是可选
max_replication_slots = 5           # max number of replication slots
# 更改wal发送最大进程数（默认值为10），这个值和上面的solts设置一样
max_wal_senders = 5    # max number of walsender processes
# 中断那些停止活动超过指定毫秒数的复制连接，可以适当设置大一点（默认60s）
wal_sender_timeout = 180s	# in milliseconds; 0 disable
```
+ 主库创建复制槽
```sql
select pg_create_logical_replication_slot('slotname','wal2json')
--创建相应用户
CREATE USER user WITH PASSWORD 'pwd';
-- 给用户复制流权限
ALTER ROLE user replication;
-- 给用户登录数据库权限
grant CONNECT ON DATABASE test to user;
-- 把当前库public下所有表查询权限赋给用户
GRANT SELECT ON ALL TABLES IN SCHEMA public TO user;
-- 设置发布为true
update pg_publication set puballtables=true where pubname is not null;
-- 把所有表进行发布
CREATE PUBLICATION dbz_publication FOR ALL TABLES;
-- 查询哪些表已经发布
select * from pg_publication_tables;
-- 更改复制标识包含更新和删除之前值
ALTER TABLE test0425 REPLICA IDENTITY FULL;
-- 查看复制标识（为f标识说明设置成功）
select relreplident from pg_class where relname='test0425';

```
+ 初始化备库
```bash
pg_basebackup -h mdw -U postgres -F p -P -R -D /data/oscg/pgpool/data1 -p 11002 -l backuplog 
```
+ 备库的postgresql.conf添加主库链接信息
```
primary_conninfo='application_name=mdw1 user=postgres host=mdw port=11002 sslmode=disable sslcompression=1 target_session_attrs=any'
```
+ 生产备库标志文件
```bash
touch /data/oscg/pgpool/data1/standby.signal

```
+ 启动备库
```bash
pg_ctl start -D data1
```

+ 主节点查看状态
```sql
select * from pg_stat_replication;
```


## 主备切换
+ 关闭主节点（模拟故障）
```bash
pg_ctl stop -D data0
```

+ 提升备节点为主节点
```bash
#方式1
pg_ctl promote -D data1
#方式2 sql
select pg_promote(true,60)
```
+ 查看数据库状态
```bash
pg_controldata -D /data/oscg/pgpool/data0|grep 'state'
#返回结果恢复模式
Database cluster state:               in archive recovery
#返回结果生产模式
Database cluster state:               in production
```

## 原主节点作为备节点添加进集群
+ 原备节点pg_hba.conf添加原主节点的链接许可
```
host    replication     all             192.168.240.14/32       trust
```
+ 原主节点添加现主节点（原备节点）的链接方式 postgresql.conf
```
primary_conninfo='application_name=mdw user=postgres host=mdw port=11002'
```
+ 创建备库标志
```bash
touch /data/oscg/pgpool/data0/standby.signal
```
## 使用pg_rewind替代pg_basebackup
> pg_rewind优势：支持增量复制，如果备节点挂掉较长时间，主节点保留的日志已经不够进行同步了，这种情况就可以使用pg_rewind进行增量同步一次数据再尝试拉起。

+ 使用前提，需要在主节点postgresql.conf设置
```
#必须设置
wal_log_hints: on
#此项默认存在
full_page_writes: on
```
## 高可用
