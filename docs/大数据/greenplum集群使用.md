# greenplum集群使用

## FDW插件使用

### mysql_fdw插件使用

```sql
#在主节点以及备份主节点用root用户执行
#初次使用需要创建扩展,在mdw用gpadmin用户source env.sh后执行
create extension mysql_fdw;
#后续使用直接创建服务器以及用户映射即可
create server bz47 foreign data wrapper mysql_fdw options (host 'hereisip' ,port '3306');
#把服务器转移给hjdev用户
ALTER SERVER "bz47" OWNER TO "hjdev";
#创建用户映射
create user mapping for hjdev server bz47 options (username 'root',password 'pass');
#创建外部表
CREATE FOREIGN TABLE "public"."out_yy_regions" (
  "indexCode" varchar(255),
  "name" varchar(255),
  "parentIndexCode" varchar(255),
  "treeCode" varchar(255),
  "id" int4
)
SERVER "bz47"
OPTIONS ("dbname" 'db', "table_name" 'yy_regions');
```

### oracle_fdw插件使用

```sql
#在主节点以及备份主节点用root用户执行
#初次使用需要创建扩展,在mdw用gpadmin用户source env.sh后执行
create extension oracle_fdw;
#后续使用直接创建服务器以及用户映射即可
create server orcljcz1 foreign data wrapper oracle_fdw options (dbserver '//123.123.29.205:1521/orcl');
#把服务器转移给hjdev用户
ALTER SERVER "orcljcz1" OWNER TO "hjdev";
#创建用户映射
create user mapping for hjdev server orcljcz1 options (user 'thieisuser',password 'thisispass');
#创建外部表
CREATE FOREIGN TABLE "public"."out_jcz1_csl_menu" (
  "id" varchar(255) NOT NULL,
  "menu_name" varchar(255),
  "ranking" varchar(255),
  "href" varchar(255),
  "icon" varchar(255),
  "pid" varchar(255),
  "is_leaf" varchar(255),
  "is_base" int4,
  "menu_key" varchar(255),
  "tag" varchar(255)
)
SERVER "orcljcz1"
OPTIONS ("schema" 'CSL', "table" 'CSL_MENU');

ALTER FOREIGN TABLE "public"."out_jcz1_csl_menu" 
  OWNER TO "hjdev";
```

## 使用技巧

### 分布字段说明

> greenplum是分布式的数据库,所以需要一个分布字段用来告诉数据库根据哪个字段来分布数据,最好使用一个自增序列作为分布键这样可以将数据均匀的分布到每个数据节点,分布键默认为第一个字段;

### 建立索引说明

> 建立索引时候选择的第一个字段在你的sql查询中必须传入该条件,否则索引用处不大严重影响查询效率,第一个之外的其他字段不受影响,实测在20个实例的gp集群中从9亿条数据里正确使用索引查询加排序耗时1秒左右(数据量6000w+)

### 备份与恢复

```bash
#只备份结构
pg_dump -U hjdev -f ./dbstruct.sql -s lh
#备份结构加数据
pg_dump -U hjdev -f ./dbdata.sql lhlh#恢复
psql -U hjdev -f ./dbdata.sql lhlh
```

## 扩容集群

- 首先按照部署文档升级系统内核以及sysctl设置还有ulimit的设置
- 安装docker并加入原有集群的网络,在原有集群执行docker swarm join-token manager然后将输出的命令在新机器执行
- 每次至少添加两台机器
- 编写扩容配置文件addhost(其中dbid是从数据库中`select * from gp_segment_configuration order by content`顺着增加的,content也是在现有基础上挨着增加的,datadir中的pgseg后的数字是取值于content

```
#<hostname>|<address>|<port>|<datadir>|<dbid>|<content>|<preferred_role>
sdw4|sdw4|10000|/home/gpadmin/data/gpseg6|15|6|p
sdw4|sdw4|10001|/home/gpadmin/data/gpseg7|16|7|p
sdw5|sdw5|10000|/home/gpadmin/data/gpseg8|19|8|p
sdw5|sdw5|10001|/home/gpadmin/data/gpseg9|20|9|p
sdw5|sdw5|30000|/home/gpadmin/mirror/gpseg6|17|6|m
sdw5|sdw5|30001|/home/gpadmin/mirror/gpseg7|18|7|m
sdw4|sdw4|30000|/home/gpadmin/mirror/gpseg8|21|8|m
sdw4|sdw4|30001|/home/gpadmin/mirror/gpseg9|22|9|m
```

- 分别在要添加的机器执行`ipcs -ls`可以在主节点直接ssh过去执行,看输出的数值是否和现有集群的一样
- 修改主节点(pgseg0)的主要实例和镜像实例的`gpseg0/pg_hba.conf`文件,在其中添加一行`host all gpadmin 192.168.123.10/24 trust` 其中 192.168.123.10是集群的虚拟ip段可通过`ip addr`获取到;
- 执行扩容(添加节点步骤):`gpexpand -i addhost -v` ,如果无意外的话节点添加成功可以通过`gpstate -a`查看也可以通过`select * from gp_segment_configuration order by content`查看
- 执行数据均衡(闲时执行):`gpexpand -d 2:00:00`代表执行最多2个小时的数据均衡

```
#输出日志(在均衡过程中的表几乎无法访问)
20210122:12:07:11:043279 gpexpand:mdw:gpadmin-[INFO]:-Querying gpexpand schema for current expansion state
20210122:12:07:12:043279 gpexpand:mdw:gpadmin-[INFO]:-Expanding lh.public.czrkjbxx
20210122:12:07:12:043279 gpexpand:mdw:gpadmin-[INFO]:-Finished expanding lhlhublic.czrkjbxx
20210122:12:07:12:043279 gpexpand:mdw:gpadmin-[INFO]:-Expanding lhlhlic.myreg
20210122:12:07:12:043279 gpexpand:mdw:gpadmin-[INFO]:-Finished expanding lh.plhc.myreg
20210122:12:07:12:043279 gpexpand:mdw:gpadmin-[INFO]:-Expanding lh.publhcar_pass
```

### 物化视图功能

- 物化视图是pg类数据库特有一项功能，意即将视图中的数据缓存的gp中，并且可以在之上建立索引以及增量更新等特性
- 创建物化视图（可以基于外部表）

```sql
create MATERIALIZED view mv_out_czrkupdate as select * from out_czrkupdate with no data;
-- 刷新数据
refresh MATERIALIZED view mv_out_czrkupdate;
-- 创建索引
create index idx_mv_outczk_pt on mv_out_czrkupdate(gmsfhm);

--指定分布键的物化视图
create MATERIALIZED view mv_out_lhhj_yhcar as select * from out_lhhj_yhcar with no data Distributed by (pid);
-- 创建唯一索引（增量刷新必备）
create index idx_mv_out_lhlh_yhcar on mv_out_lhhj_yhcar(pid);
-- 全量刷新
-- 增量刷新（增量刷新前必须全量刷新一次）
refresh MATERIALIZED view concurrently mv_out_lhhj_yhcar;
```

## 故障恢复

### 恢复所有有问题的节点(节点的容器启动状态下)

```
#gpmaster gpadmin登录
source env.sh
gprecoverseg --hba-hostnames -a
#执行成功上面命令后通过下面命令查看(上面命令执行成功后需要等等三五分钟或者通过`gpstate -s`查看同步状态)
gpstate -e
#如果始终有节点错误就继续使用下面全量同步模式恢复节点
gprecoverseg --hba-hostnames -F -a
#成功完成后执行
gprecoverseg ---hba-hostnames -r -a
#最后通过下面命令查看(上面命令执行成功后需要等等三五分钟或者通过`gpstate -s`查看同步状态)
gpstate -e
```

### 自动清理过期日志

> 从容器外面通过docker exec xxx来调用

```
#!/bin/bash
cd /home/gpadmin/master
logf="perday.log"
echo "begin check gpstate `date` " > $logf
source /home/gpadmin/env.sh
gprecoverseg --hba-hostnames -a >> $logf
gprecoverseg --hba-hostnames -F -a >> $logf
gprecoverseg --hba-hostnames -r -a >> $logf
gpstate -a >> $logf
#clean old log
find /home/gpadmin -name "gpdb-*.csv" -atime +3 |xargs rm -f
echo mdw >> $logf
df -h >> $logf
gpssh -f /home/gpadmin/hostfile -e 'df -h' >> $logf
gpssh -f /home/gpadmin/hostfile -e 'find /home/gpadmin -name "gpdb-*.csv" -atime +3 |xargs rm -f' >> $logf
echo "check over" >> $logf
```

### 维护模式

- 维护模式启动:`gpstart -m`
- 维护模式连接:`PGOPTIONS='-c gp_session_role=utility' psql dbname`
- 维护完毕正常启动:`gpstop -mr`

## sql技巧

### 对某个表进行统计并查看

> 此方法可以根据数据表结构和存储的数据分析出具体每列的最常见值、出现频率、null率等常用统计信息，可以很方便的制作热力图一类 注意：对于数据量较多的表，此方法统计出的信息只是抽样统计不是全量统计！

```
analyze car_pass;
select * from pg_stats where tablename='car_pass';
```

### 查询当前连接数

```
select count(1) from pg_stat_activity;
```

## 注意事项

- 严禁使用kill命令杀postgres进程，有类似需要请使用数据库命令`pg_terminate_backend()` 来完成
- 每天定时analyze以及vacuum大表或者数据量每天变化较大的表

## greenplum关闭现有连接

> 当普通用户无法连接数据库，提示连接过多时候我们通过管理员模式进去查出所有连接，然后进行关闭即可

```
#使用超管模式进入
PGOPTIONS='-c gp_session_role=utility' psql test
#如果要关闭所有连接
select pg_terminate_backend(pid) from pg_stat_activity;
#如果关闭连接到某个数据库的连接
select pg_terminate_backend(pid) from pg_stat_activity where datname='test';
```

## 列式存储特点

### 优势

- 数据按列存储可以获得较高的压缩比
- 当查询少量字段时候，扫描的块更少，节约iO且提升效率

### 劣势

- 查询大量字段或查询记录数偏少时会造成较多离散IO
- 由于IO放大，列存储不适合OLTP场景，如有大量更新查询操作的表

### 适用场景

- 典型的OLAP场景、按列做较大范围的聚合分析或者JOIN分析
- 如果用户频繁的装载和更新表数据适用于面向行的heap表，面向列适用于追加优化的表（写入多）
- 频繁insert整行的请选择面向追加优化的行存储表
- 查询中经常需要返回全部或者大部分列的情况下请使用面向行存储的方式
- where条件单一且返回较少行，适用于列存
- 列存支持压缩，行存不支持

## 追加优化表使用

### 列式追加表

```
create table tac (id serial4,t1 varchar(255) with(appendonly=true,orientation=column) distributed by (id) ;
```

### 行式追加表

```
create table tarc (id serial4,t1 varchar(255) with(appendonly=true,orientation=row) distributed by (id) ;
```

### 普通行式heap表

```
create table tarc (id serial4,t1 varchar(255)  distributed by (id) ;
```

## 死锁排查

```
select * from gp_toolkit.gp_locks_on_relation limit 10
--select pg_terminate_backend(xxx)
```