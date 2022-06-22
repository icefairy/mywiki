# Flink-sql-ETL集合
# 名词解释
+ cdc: 变动数据抓取 机制是flink使用或者发扬光大的的一种数全量或增量同步数据的机制，支持各种常见数据库的变动获取，例如mysql、postgresql、oracle等，一般需要对数据库系统做一定的配置，例如开启binlog、wal日志、redo日志等。且cdc作为变动抓取的工具一般只支持读取，所以带cdc字样的connector都只能作为数据的输入源。

## 通用
+ save-point机制
```sql
SET 'execution.savepoint.path' = '/data/flink/checkpoint/111111';
```
## mysql-cdc
+ 前置条件：mysqld.cnf中开启binglog并进行一定设置，然后重启数据库：
```
#大于0的正整数
server_id=123
#必须
log_bin=mysql-bin
#必须
binlog_format=row
#日志保留最少2天
expire_logs_days=7
#可以只对指定的数据库开启binlog 这下面是可选的设置项
binlog_do_db=xxxdb
#如果要对xxxdb2开启则再加一行
binlog_do_db=xxxdb2
```
+ 同步的用户如果不是root则需要给权限：
```sql
grant replication slave on *.* to youruser;
grant replication client on *.* to youruser;
flush PRIVILEGES;
```
+ mysql-cdc 全量参数列表：https://ververica.github.io/flink-cdc-connectors/release-2.2/content/connectors/mysql-cdc.html
+ 使用对应版本的mysql-cdc-connector的抽取样例：
```sql
set execution.checkpointing.interval=3s;
--设置并发度（比如多个任务从kafka接收数据）
set 'execution.parallelism'='5';
create table sender(id int,code string,name string,primary key(id) not enforced) with ('connector'='mysql-cdc','hostname'='192.168.240.14','port'='3306','username'='root','password'='password','database-name'='dragon','table-name'='sender','server-id'=1000);
--注意输出用的jdbc connector 需要自己在目标库建好相应的表，否则会报错
create table sender2(id int,code string,name string,primary key(id) not enforced) with ('connector'='jdbc','url'='jdbc:mysql://192.168.240.14:3306/dragon','table-name'='sender2','username'='root','password'='password');
--给任务命名
SET 'pipeline.name' = 'yhface2db';
insert into sender2  select id,code,name from sender;

```

## postgresql-cdc
+ 前置条件：在postgres.conf中开启wal日志，并设置wal_level相关参数
```
# 更改wal日志方式为logical 必须
wal_level = logical            # minimal, replica, or logical
# 更改solts最大数量（默认值为10），flink-cdc默认一张表占用一个slots 下面开始是可选
max_replication_slots = 20           # max number of replication slots
# 更改wal发送最大进程数（默认值为10），这个值和上面的solts设置一样
max_wal_senders = 20    # max number of walsender processes
# 中断那些停止活动超过指定毫秒数的复制连接，可以适当设置大一点（默认60s）
wal_sender_timeout = 180s	# in milliseconds; 0 disable
```
+ 给用户设置权限
```sql
-- pg新建用户
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
+ postgres同步到es使用样例
```sql
set execution.checkpointing.interval=3s;
create table pgck(pid String,id String,gmsfhm String,xm String,csrq String,xbdm String,xbmc String,mzdm String,mzmc String,xxdm String,xxmc String,jggjdm String,jggjmc String,jgssxdm String,jgssxmc String,csdjgdm String,csdjgmc String,csdssxdm String,csdssxmc String,csdxz String,hjdpcsdm String,hjdpcsmc String,hjdxz String,xzz String,lxdh String,zzmmdm String,zzmmmc String,zw String,zy String,byzkdm String,byzkmc String,whcddm String,whcdmc String,hyzkdm String,hyzkmc String,zjxydm String,zjxymc String,hklxdm String,hklxmc String,hkxzdm String,hkxzmc String,hh String,yhzgxdm String,yhzgxmc String,fqgmsfhm String,fqxm String,mqgmsfhm String,mqxm String,pogmsfhm String,poxm String,idcard_url String,province String,city String,country string,primary key(pid) not enforced) with ('connector'='postgres-cdc','hostname'='192.168.240.14','port'='5432','database-name'='demo','schema-name'='public','username'='postgres','password'='pass','table-name'='czrkjbxx','decoding.plugin.name'='pgoutput');

create table czrk(pid String,id String,gmsfhm String,xm String,csrq String,xbdm String,xbmc String,mzdm String,mzmc String,xxdm String,xxmc String,jggjdm String,jggjmc String,jgssxdm String,jgssxmc String,csdjgdm String,csdjgmc String,csdssxdm String,csdssxmc String,csdxz String,hjdpcsdm String,hjdpcsmc String,hjdxz String,xzz String,lxdh String,zzmmdm String,zzmmmc String,zw String,zy String,byzkdm String,byzkmc String,whcddm String,whcdmc String,hyzkdm String,hyzkmc String,zjxydm String,zjxymc String,hklxdm String,hklxmc String,hkxzdm String,hkxzmc String,hh String,yhzgxdm String,yhzgxmc String,fqgmsfhm String,fqxm String,mqgmsfhm String,mqxm String,pogmsfhm String,poxm String,idcard_url String,province String,city String,country string,primary key(pid) not enforced) with ('connector'='elasticsearch-6','hosts'='http://192.168.240.14:9200','index'='czrk','document-type'='czrk','username'='','password'='');

insert into czrk select pid,id,gmsfhm,xm,csrq,xbdm,xbmc,mzdm,mzmc,xxdm,xxmc,jggjdm,jggjmc,jgssxdm,jgssxmc,csdjgdm,csdjgmc,csdssxdm,csdssxmc,csdxz,hjdpcsdm,hjdpcsmc,hjdxz,xzz,lxdh,zzmmdm,zzmmmc,zw,zy,byzkdm,byzkmc,whcddm,whcdmc,hyzkdm,hyzkmc,zjxydm,zjxymc,hklxdm,hklxmc,hkxzdm,hkxzmc,hh,yhzgxdm,yhzgxmc,fqgmsfhm,fqxm,mqgmsfhm,mqxm,pogmsfhm,poxm,idcard_url,province,city,country from pgck;
```

## pg到starrock
```sql
--先在starrocks建表
create table czrkjbxx(pid bigint,id String,gmsfhm String,xm String,csrq String,xbdm String,xbmc String,mzdm String,mzmc String,xxdm String,xxmc String,jggjdm String,jggjmc String,jgssxdm String,jgssxmc String,csdjgdm String,csdjgmc String,csdssxdm String,csdssxmc String,csdxz String,hjdpcsdm String,hjdpcsmc String,hjdxz String,xzz String,lxdh String,zzmmdm String,zzmmmc String,zw String,zy String,byzkdm String,byzkmc String,whcddm String,whcdmc String,hyzkdm String,hyzkmc String,zjxydm String,zjxymc String,hklxdm String,hklxmc String,hkxzdm String,hkxzmc String,hh String,yhzgxdm String,yhzgxmc String,fqgmsfhm String,fqxm String,mqgmsfhm String,mqxm String,pogmsfhm String,poxm String,idcard_url String,province String,city String,country string) primary key(pid)  distributed by hash(pid) properties("replication_num"="1")
--下面的语句在flink-sql中执行
set execution.checkpointing.interval=3s;
create table pgck(pid bigint,id String,gmsfhm String,xm String,csrq String,xbdm String,xbmc String,mzdm String,mzmc String,xxdm String,xxmc String,jggjdm String,jggjmc String,jgssxdm String,jgssxmc String,csdjgdm String,csdjgmc String,csdssxdm String,csdssxmc String,csdxz String,hjdpcsdm String,hjdpcsmc String,hjdxz String,xzz String,lxdh String,zzmmdm String,zzmmmc String,zw String,zy String,byzkdm String,byzkmc String,whcddm String,whcdmc String,hyzkdm String,hyzkmc String,zjxydm String,zjxymc String,hklxdm String,hklxmc String,hkxzdm String,hkxzmc String,hh String,yhzgxdm String,yhzgxmc String,fqgmsfhm String,fqxm String,mqgmsfhm String,mqxm String,pogmsfhm String,poxm String,idcard_url String,province String,city String,country string,primary key(pid) not enforced) with ('connector'='postgres-cdc','hostname'='192.168.240.14','port'='5432','database-name'='demo','schema-name'='public','username'='postgres','password'='pass','table-name'='czrkjbxx','decoding.plugin.name'='pgoutput');

--starrocks作为输出时候选项中有load-url作为输入时候将load-url换成scan-url,输出时候需要自建表
create table czrksr(pid bigint,id String,gmsfhm String,xm String,csrq String,xbdm String,xbmc String,mzdm String,mzmc String,xxdm String,xxmc String,jggjdm String,jggjmc String,jgssxdm String,jgssxmc String,csdjgdm String,csdjgmc String,csdssxdm String,csdssxmc String,csdxz String,hjdpcsdm String,hjdpcsmc String,hjdxz String,xzz String,lxdh String,zzmmdm String,zzmmmc String,zw String,zy String,byzkdm String,byzkmc String,whcddm String,whcdmc String,hyzkdm String,hyzkmc String,zjxydm String,zjxymc String,hklxdm String,hklxmc String,hkxzdm String,hkxzmc String,hh String,yhzgxdm String,yhzgxmc String,fqgmsfhm String,fqxm String,mqgmsfhm String,mqxm String,pogmsfhm String,poxm String,idcard_url String,province String,city String,country string,primary key(pid) not enforced) with ('connector'='starrocks','load-url'='192.168.125.16:8030','jdbc-url'='jdbc:mysql://192.168.125.16:9030','username'='root','password'='','database-name'='lhcz','table-name'='czrkjbxx');

insert into czrksr select pid,id,gmsfhm,xm,csrq,xbdm,xbmc,mzdm,mzmc,xxdm,xxmc,jggjdm,jggjmc,jgssxdm,jgssxmc,csdjgdm,csdjgmc,csdssxdm,csdssxmc,csdxz,hjdpcsdm,hjdpcsmc,hjdxz,xzz,lxdh,zzmmdm,zzmmmc,zw,zy,byzkdm,byzkmc,whcddm,whcdmc,hyzkdm,hyzkmc,zjxydm,zjxymc,hklxdm,hklxmc,hkxzdm,hkxzmc,hh,yhzgxdm,yhzgxmc,fqgmsfhm,fqxm,mqgmsfhm,mqxm,pogmsfhm,poxm,idcard_url,province,city,country from pgck;

```
## pg2pg
```sql
create table sender2(id int,code string,name string,primary key(id) not enforced) with ('connector'='jdbc','url'='jdbc:mysql://192.168.240.14:3306/dragon','table-name'='sender2','username'='root','password'='password');
```

##　kafka-connector解析复杂数据
```json
{
    "afterColumns":{
        "created":"1589186680",
        "extra":{
            "canGiving":false
        },
        "parameter":[
            1,
            2,
            3,
            4
        ]
    },
    "beforeColumns":null,
    "tableVersion":{
        "binlogFile":null,
        "binlogPosition":0,
        "version":0
    },
    "touchTime":1589186680591
}
```
```sql
--创建解析表
CREATE TABLE t1 ( 
	afterColumns ROW(created STRING,extra ROW(canGiving BOOLEAN),`parameter` ARRAY <INT>) ,
	beforeColumns STRING ,
	tableVersion ROW(binlogFile STRING,binlogPosition INT ,version INT) ,
	touchTime bigint 
	) WITH (
	    'connector.type' = 'kafka',  -- 指定连接类型是kafka
	    'connector.version' = '0.11',  -- 与我们之前Docker安装的kafka版本要一致
	    'connector.topic' = 'json_parse', -- 之前创建的topic 
	    'connector.properties.group.id' = 'flink-test-0', -- 消费者组，相关概念可自行百度
	    'connector.startup-mode' = 'earliest-offset',  --指定从最早消费
	    'connector.properties.zookeeper.connect' = 'localhost:2181',  -- zk地址
	    'connector.properties.bootstrap.servers' = 'localhost:9092',  -- broker地址
	    'format.type' = 'json'  -- json格式，和topic中的消息格式保持一致
	)
	这里我们注意一下，parameter得用``括起来，因为parameter是个关键字，想用的话得用``包围，所有关键字可以参考https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/table/sql/#reserved-keywords
	最后输出的数据是这样的：1589186680,false,[1, 2, 3, 4],null,null,0,0,1589186680591，和我们JSON中的数据一样，没毛病！
	不过我们用的是select *，正常来说我们应该只输出想要的字段。那么，假设我们只想拿到canGiving 这个嵌套好几层的字段值，该怎么写呢？
	其实很简单，就像是包路径一样 afterColumns.extra.canGiving ，试一下
	select  afterColumns.extra.canGiving from t1
	输出：false
	再试一下输出 parameter 这个集合的第一个值(下标从1开始)
	select afterColumns.`parameter`[1] from t1
```