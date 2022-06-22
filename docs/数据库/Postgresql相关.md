## 常用配置

### max_connections

允许的最大客户端连接数。这个参数设置大小和work_mem有一些关系。配置的越高，可能会占用系统更多的内存。通常可以设置数百个连接，如果要使用上千个连接，建议配置连接池来减少开销。

### shared_buffers

PostgreSQL使用自己的缓冲区，也使用Linux操作系统内核缓冲OS Cache。这就说明数据两次存储在内存中，首先是PostgreSQL缓冲区，然后是操作系统内核缓冲区。与其他数据库不同，PostgreSQL不提供直接IO，所以这又被称为双缓冲。PostgreSQL缓冲区称为shared_buffer，建议设置为物理内存的1/4。而实际配置取决于硬件配置和工作负载，如果你的内存很大，而你又想多缓冲一些数据到内存中，可以继续调大shared_buffer。

### Effective_cache_size

这个参数主要用于Postgre查询优化器。是单个查询可用的磁盘高速缓存的有效大小的一个假设，是一个估算值，它并不占据系统内存。由于优化器需要进行估算成本，较高的值更有可能使用索引扫描，较低的值则有可能使用顺序扫描。一般这个值设置为内存的1/2是正常保守的设置，设置为内存的3/4是比较推荐的值。通过free命令查看操作系统的统计信息，您可能会更好的估算该值。

```
[pg@e22 ~]$ free -g
              total        used        free      shared  buff/cache   available
Mem:             62           2           5          16          55          40
Swap:             7           0           7
```

### work_mem

这个参数主要用于写入临时文件之前内部排序操作和散列表使用的内存量，增加work_mem参数将使PostgreSQL可以进行更大的内存排序。这个参数和max_connections有一些关系，假设你设置为30MB，则40个用户同时执行查询排序，很快就会使用1.2GB的实际内存。同时对于复杂查询，可能会运行多个排序和散列操作，例如涉及到8张表进行合并排序，此时就需要8倍的work_mem。

如下面案例所示，该环境使用4MB的work_mem，在执行排序操作的时候，使用的Sort Method是external merge Disk。

```
external merge Disk。

kms=> explain (analyze,buffers) select * from KMS_BUSINESS_HALL_TOTAL  order by buss_query_info;
                                                                       QUERY PLAN                                                                      
---------------------------------------------------------------------------------------------------------------------------------------------------------
 Gather Merge  (cost=262167.99..567195.15 rows=2614336 width=52) (actual time=2782.203..5184.442 rows=3137204 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   Buffers: shared hit=68 read=25939, temp read=28863 written=28947
   ->  Sort  (cost=261167.97..264435.89 rows=1307168 width=52) (actual time=2760.566..3453.783 rows=1045735 loops=3)
         Sort Key: buss_query_info
         Sort Method: external merge  Disk: 50568kB
         Worker 0:  Sort Method: external merge  Disk: 50840kB
         Worker 1:  Sort Method: external merge  Disk: 49944kB
         Buffers: shared hit=68 read=25939, temp read=28863 written=28947
         ->  Parallel Seq Scan on kms_business_hall_total  (cost=0.00..39010.68 rows=1307168 width=52) (actual time=0.547..259.524 rows=1045735 loops=3)
               Buffers: shared read=25939
 Planning Time: 0.540 ms
 Execution Time: 5461.516 ms
(14 rows)
```

当我们把参数修改成512MB的时候，可以看到Sort Method变成了quicksort Memory，变成了内存排序。

```
kms=> set work_mem to "512MB";
SET
kms=> explain (analyze,buffers) select * from KMS_BUSINESS_HALL_TOTAL  order by buss_query_info;
                                                                QUERY PLAN                                                              
------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=395831.79..403674.80 rows=3137204 width=52) (actual time=7870.826..8204.794 rows=3137204 loops=1)
   Sort Key: buss_query_info
   Sort Method: quicksort  Memory: 359833kB
   Buffers: shared hit=25939
   ->  Seq Scan on kms_business_hall_total  (cost=0.00..57311.04 rows=3137204 width=52) (actual time=0.019..373.067 rows=3137204 loops=1)
         Buffers: shared hit=25939
 Planning Time: 0.081 ms
 Execution Time: 8419.994 ms
(8 rows)
```

### maintenance_work_mem

指定维护操作使用的最大内存量，例如（Vacuum、Create Index和Alter Table Add Foreign Key），默认值是64MB。由于通常正常运行的数据库中不会有大量并发的此类操作，可以设置的较大一些，提高清理和创建索引外键的速度。

```
postgres=# set maintenance_work_mem to "64MB";
SET
Time: 1.971 ms
postgres=# create index idx1_test on test(id);
CREATE INDEX
Time: 7483.621 ms (00:07.484)
postgres=# set maintenance_work_mem to "2GB";
SET
Time: 0.543 ms
postgres=# drop index idx1_test;
DROP INDEX
Time: 133.984 ms
postgres=# create index idx1_test on test(id);
CREATE INDEX
Time: 5661.018 ms (00:05.661)
```

可以看到在使用默认的64MB创建索引，速度为7.4秒，而设置为2GB后，创建速度是5.6秒

### wal_sync_method

每次发生事务后，PostgreSQL会强制将提交写到WAL日志的方式。可以使用pg_test_fsync命令在你的操作系统上进行测试，fdatasync是Linux上的默认方法。如下所示，我的环境测试下来fdatasync还是速度可以的。不支持的方法像fsync_writethrough直接显示n/a。

```
postgres=# show wal_sync_method ;
 wal_sync_method 
-----------------
 fdatasync
(1 row)

[pg@e22 ~]$ pg_test_fsync -s 3
3 seconds per test
O_DIRECT supported on this platform for open_datasync and open_sync.

Compare file sync methods using one 8kB write:
(in wal_sync_method preference order, except fdatasync is Linux's default)
        open_datasync                      4782.871 ops/sec     209 usecs/op
        fdatasync                          4935.556 ops/sec     203 usecs/op
        fsync                              3781.254 ops/sec     264 usecs/op
        fsync_writethrough                              n/a
        open_sync                          3850.219 ops/sec     260 usecs/op

Compare file sync methods using two 8kB writes:
(in wal_sync_method preference order, except fdatasync is Linux's default)
        open_datasync                      2469.646 ops/sec     405 usecs/op
        fdatasync                          4412.266 ops/sec     227 usecs/op
        fsync                              3432.794 ops/sec     291 usecs/op
        fsync_writethrough                              n/a
        open_sync                          1929.221 ops/sec     518 usecs/op

Compare open_sync with different write sizes:
(This is designed to compare the cost of writing 16kB in different write
open_sync sizes.)
         1 * 16kB open_sync write          3159.780 ops/sec     316 usecs/op
         2 *  8kB open_sync writes         1944.723 ops/sec     514 usecs/op
         4 *  4kB open_sync writes          993.173 ops/sec    1007 usecs/op
         8 *  2kB open_sync writes          493.396 ops/sec    2027 usecs/op
        16 *  1kB open_sync writes          249.762 ops/sec    4004 usecs/op

Test if fsync on non-write file descriptor is honored:
(If the times are similar, fsync() can sync data written on a different
descriptor.)
        write, fsync, close                3719.973 ops/sec     269 usecs/op
        write, close, fsync                3651.820 ops/sec     274 usecs/op

Non-sync'ed 8kB writes:
        write                            400577.329 ops/sec       2 usecs/op
```

### wal_buffers

事务日志缓冲区的大小，PostgreSQL将WAL记录写入缓冲区，然后再将缓冲区刷新到磁盘。在PostgreSQL 12版中，默认值为-1，也就是选择等于shared_buffers的1/32 。如果自动的选择太大或太小可以手工设置该值。一般考虑设置为16MB。

### synchronous_commit

客户端执行提交，并且等待WAL写入磁盘之后，然后再将成功状态返回给客户端。可以设置为on，remote_apply，remote_write，local，off等值。默认设置为on。如果设置为off，会关闭sync_commit，客户端提交之后就立马返回，不用等记录刷新到磁盘。此时如果PostgreSQL实例崩溃，则最后几个异步提交将会丢失。

### default_statistics_target

PostgreSQL使用统计信息来生成执行计划。统计信息可以通过手动Analyze命令或者是autovacuum进程启动的自动分析来收集，default_statistics_target参数指定在收集和记录这些统计信息时的详细程度。默认值为100对于大多数工作负载是比较合理的，对于非常简单的查询，较小的值可能会有用，而对于复杂的查询（尤其是针对大型表的查询），较大的值可能会更好。为了不要一刀切，可以使用ALTER TABLE .. ALTER COLUMN .. SET STATISTICS覆盖特定表列的默认收集统计信息的详细程度。

### checkpoint_timeout、max_wal_size，min_wal_size、checkpoint_completion_target

了解这两个参数以前，首先我们来看一下，触发检查点的几个操作。

直接执行checkpoint命令 执行需要检查点的命令（例如pg_start_backup,Create database,pg_ctl stop/start等等） 自上一个检查点以来，达到了已经配置的时间量（checkpoint_timeout ） 自上一个检查点以来生成的WAL数量（max_wal_size） 使用默认值，检查点将在checkpoint_timeout=5min。也就是每5分钟触发一次。而max_wal_size设置是自动检查点之间增长的最大预写日志记录（WAL）量。默认是1GB，如果超过了1GB，则会发生检查点。这是一个软限制。在一个特殊的情况下，比如系统遭遇到短时间的高负载，日志产生几秒种就可以达到1GB，这个速度已经明显超过了checkpoint_timeout ，pg_wal目录的大小会急剧增加。此时我们可以从日志中看到相关类似的警告。

```
LOG:  checkpoints are occurring too frequently (9 seconds apart)
HINT:  Consider increasing the configuration parameter "max_wal_size".
LOG:  checkpoints are occurring too frequently (2 seconds apart)
HINT:  Consider increasing the configuration parameter "max_wal_size".
```

所以要合理配置max_wal_size，以避免频繁的进行检查点。一般推荐设置为16GB以上，不过具体设置多大还需要和工作负荷相匹配。

min_wal_size参数是只要 WAL 磁盘使用量保持在这个设置之下，在做检查点时，旧的 WAL 文件总是被回收以便未来使用，而不是直接被删除。

而检查点的写入不是全部立马完成的，PostgreSQL会将一次检查点的所有操作分散到一段时间内。这段时间由参数checkpoint_completion_target控制，它是一个分数，默认为0.5。也就是在两次检查点之间的0.5比例完成写盘操作。如果设置的很小，则检查点进程就会更加迅速的写盘，设置的很大，则就会比较慢。一般推荐设置为0.9，让检查点的写入分散一点。但是缺点就是出现故障的时候，影响恢复的时间。

可以输入参数使用pgtune自动生成配置建议：https://pgtune.leopard.in.ua/#/

## 外部表

### mysql/mariadb/tidb外部表

```sql
--举例中本地数据库用户为user
--创建fdw插件（如容器镜像pg13命令为apt-get update && apt-get install -y postgresql-13-mysql-fdw)
create extension mysql_fdw;
--注册外部服务器
create server outdb foreign data wrapper mysql_fdw options (host '123.123.123.123' ,port '3306');
--在greenplum中，如果上面命令用pgadmin用户执行的话，需要将所有权给到普通用户
ALTER SERVER "outdb" OWNER TO "user";
--创建用户映射
create user mapping for user server outdb options (username 'root',password 'password');
--创建外部表
CREATE FOREIGN TABLE "public"."out_yy_regions" (
  "indexCode" varchar(255),
  "name" varchar(255),
  "parentIndexCode" varchar(255),
  "treeCode" varchar(255),
  "id" int4
)
SERVER "outdb"
OPTIONS ("dbname" 'mydb', "table_name" 'mytable');
```

### oracle外部表

* todo

## 物化视图

> postgres中物化视图可以将需要一定计算量或者数据传输（例如外部表）耗时的视图仿真成为一个数据表，再辅以定时刷新或者按需刷新的机制来便捷的使用该数据；
>

### 创建物化视图

```sql
--假设存在一个外部表 out_tab1其中有100万数据，如果不做物化视图的话每次查询都会去原始数据库中拿这100万的数据，对于一些对耗时敏感的操作十分不友好，此时通过物化视图来进行优化；
--创建空的物化视图
create MATERIALIZED view mv_out_tab1 as select * from out_tab1 with no data;
--如果是greenplum还需要增加分布键的声明，例如：create MATERIALIZED view mv_out_tab1 as select * from out_tab1 with no data Distributed by (id)
--刷新数据
refresh MATERIALIZED view mv_out_tab1;
```

## 常用命令
### 备份数据库或者结构
```bash
-- 备份表结构
pg_dump -h ${host} -p ${port} -d ${dbname} -U ${username} -s > /data/${dbname}.sql

-- 只备份数据 -a ，只备份结构 -s，表结构和数据 什么都不加

-- 还原库
psql -h ${host} -p ${port} -U ${username} -d ${dbname} < /data/${dbname}.sql

-- 备份指定表(包含数据)
pg_dump -h ${host} -p ${port} -d ${dbname} -t ${tablename1} -t ${tablename2}  -U ${username} > /data/${filename}.sql

-- 导出导入数据到txt

copy ${tablename} to '/data/pgdata/csvbak/xxx.txt'

copy ${tablename} from '/data/pgdata/csvbak/xxx.txt'
```

## 常用技巧


* domain 创建类型别名

```
create domain amount as decimal(33, 18);
create domain price as decimal(33, 18);
```

* 自定义枚举类型 使用的时候可以传字符串，实际存储是整数。 create type 还可以创建其他更复杂的类型，这里就不详细介绍了。create type side as enum ('buy', 'sell');
* Heap-Only-Tuple 和 fillfactor 这个和PG的MVCC实现有关系，简单的说，PG的表数据是存在无序的堆上的，包括主键的所有索引都是单独的，索引引用数据行在堆上的位置。 加上MVCC的实现，Update其实是Insert+Delete，新的行数据在堆上的位置发生改变，使得需要更新索引数据。为了优化这种情况， 如果在该条数据相同的Page上面有空闲位置，新插入数据优先放在相同页上，并和旧版本数据建立链表。在查询数据的时候，根据这个链表再去处理事务可见性等判断。 这样就不需要去更新索引数据了。如果Page上没有空闲位置就没办法了。所以 create table 的时候可以指定 fillfactor 选项，预先留出一些位置。 适合数据量不大且更新频繁的场景。
* 用snowflake算法生成唯一ID很实用，但是在PG上有个问题，PG遵循SQL标准，不支持无符号64位整数。snowflake留了41bit给毫秒级时间戳，去一位给符号位，剩下40位。 算一下发现2004年就溢出了，其实就算无符号64位整数，2039年也就溢出了。解决方案，时间戳统一减去一个最近的时间点，瞬间多出50年。 在PG里面用这种ID，结合自定义函数，连时间戳字段都省了(PG12貌似要支持虚拟字段了)。自动分表也可以按照ID进行，达到按时间戳分表一样的效果，还能保留主键功能，一举多得。
* 批量insert/update/upsert/delete/move

```sql
#数据库里面，批量操作的吞吐量和单条执行不在一个数量级。insert批量大家都知道：

insert into table values (1,3,3), (2,3,3), (3,3,3)...
#用psycopg2的朋友需要注意的是， cur.executemany 函数并不是真批量操作，他还是生成一堆独立的insert执行，真正的批量操作要这么写：
#postgresql批量操作
import time
import psycopg2
import psycopg2.extras
db=psycopg2.connect("dbname=postgres user=postgres password=pwd host=127.0.0.1")
cur=db.cursor()
#insert_values
lst=[]
for i in range(100_0000):
    lst.append(['name%s' % i,'text%s' % i])
cur.execute('truncate table ctest')
ta=time.time()
psycopg2.extras.execute_values(cur,"""
insert into ctest(uname,text) values %s""",lst)
cur.execute("commit")
print('use time:%s' % (time.time()-ta))
# use time:15.339413166046143
cur.execute("select count(1) cnt from ctest")
data=cur.fetchone()
print(data)

db.close()
#update 如何批量呢：

with tmp(id, name) as (values (1, 'a'), (2, 'b'), (3, 'c'))
update table set name=tmp.name from tmp where table.id=tmp.id;
upsert:

insert into table values (1,3,3), (2,3,3), (3,3,3)...
on conflict (a) do update set a=table.a+excluded.a
delete:

delete from table where id = ANY[%s]
#move, 有时候我们想把数据批量从一个表移动到另一个表：

with moved_rows as (
  with tmp(id, a, b) as (values %s)
  delete from t_old_table a
  using tmp
  where tmp.id=a.id
  returning a.*
)
insert into t_new_table select * from moved_rows
```

* mvcc snapshot。问题：如何把一个并发访问的表一致的分割成多段，比如我们想要分批次对里面的数据进行处理，但不希望漏数据，也不希望重复处理。

> 注解 自增ID？在并发访问下，自增ID并不能保证连续没有间断的，比如中间可能还有事务没有提交。
>

> 解决方案是，在表里保存 txid_current() ，要分段的时候，记录下 txid_current_snapshot() ，snapshot由三个信息组成：xmin, xmax, xip。 xmin是当前进行中的事务ID中最小的，也就是比xmin更小的事务都已提交或者撤销，xmax是下一个将要分配的事务ID，比xmax更大的事务还没出现，xip是当前进行中的事务ID列表。 那么一条记录是否发生在这个snapshot之前就很明确了， txid < xmin or (txid < xmax and txid not in xip) ，对应的函数是 txid_visible_in_snapshot(txid, snapshot) 。
>

* pipelinedb，流式聚合

```sql
create foreign table trade (time timestamp with time zone, price decimal, amount decimal) server pipelinedb;
create view kline_1m as select
    date_trunc('minute', time) as time,
    keyed_min(time, price) as open,
    max(price) as high,
    min(price) as low,
    keyed_max(time, price) as close,
    sum(amount) as volume,
    sum(amount * price) as value,
    count(*) as count
from trade group by 1;
create unique index kline_1m_time_idx on kline_1m(time);
然后只管疯狂对 trade 表insert， kline_1m 会自动更新聚合数据，而且 trade 不会保留数据。
```

* timescaledb，时序数据库，按照时间戳字段全自动分表。虽然pg有了声明式分表，但还是不如全自动来的爽。再也不用担心表数据量过太多影响性能了。

### large in优化

> Large In:查询条件中有例如 `id in (1,2,3……)` 很多很多值的list的情况，当传入值达到上万后效率很低。 常用方式：
>

* in-list：`where id in (1,2,3……)`
* 常量子查询: `location in (values ('晨市'),('磊市'),('静市')) limit 100`
* 临时表查询（cte）：

```sql
with tmp_ids as (
select unnest('{2100698,2100699,2100700,2100701,2100702,2100703,2100704,2100705,2100706,2100707,2100708,2100709,2100710,2100711,2100712,2100713,2100714,2100715,2100716}'::bigint[]) id
)
select * from czrkjbxx ck,tmp_ids tids where ck.pid =tids.id
```

* 实测 ：1000个值差距不明显，在达到10000个值之后，IN-list需要18秒左右，常量子查询需要4秒，临时表只需要347 ms。临时表的效率是最高的。

### CTE语法

```sql
--#同行人粗略版
SELECT FLOOR
	( EXTRACT ( epoch FROM pass_time ) ) :: int8 / 300 tick,
	pass_time,
	equipment_id 
FROM
	person_pass 
WHERE
	pass_time BETWEEN '2021-07-01 00:00:00' 
	AND '2021-07-09 23:59:59' 
GROUP BY
	FLOOR ( EXTRACT ( epoch FROM pass_time ) ) :: int8 / 300,
	pass_time,
	equipment_id 
	LIMIT 10000 
--交叠
	SELECT
	ta.NAME na,
	ta.idcard ia,
	tb.NAME nb,
	tb.idcard ib 
FROM
	( SELECT * FROM person_pass WHERE equipment_id = '652823058706422' AND ( FLOOR ( EXTRACT ( epoch FROM pass_time ) ) :: int8 / 300 ) = 5418720 ) ta
	CROSS JOIN ( SELECT * FROM person_pass WHERE equipment_id = '652823058706422' AND ( FLOOR ( EXTRACT ( epoch FROM pass_time ) ) :: int8 / 300 ) = 5418720 ) tb 
WHERE
	ta.idcard != tb.idcard
```

## 锁处理

```sql
--查询当前连接信息
select * from pg_stat_activity
--查询锁信息
select * from gp_toolkit.gp_locks_on_relation
--查询排他锁
 select * from gp_toolkit.gp_locks_on_relation where lormode not like '%Share%'
--根据pid取消某个查询（pid来自于gp_locks_on_relation 中的lorpid字段）
select pg_cancel_backend(lorpid) from gp_toolkit.gp_locks_on_relation
--根据pid强制结束某个查询（pid来自于gp_locks_on_relation 中的lorpid字段）
select pg_terminate_backend(lorpid) from gp_toolkit.gp_locks_on_relation
--强制结束所有排他锁的查询
select pg_terminate_backend(lorpid) from gp_toolkit.gp_locks_on_relation where lormode not like '%Share%' and lorpid<>pg_backend_pid()
```

## PG数据量限制

```sql
最大数据库大小无限制 

单个表最大大小 32 TB （对于block_size 8k，深层次原因是每个block都有个blocknumber，而blocknumber是个uint32类型）

最大行大小 1.6 TB 

最大字段大小 1 GB 

每张表的最大行数无限制 

每个表的最大列数 250 - 1600 取决于列类型 

每张表的最大索引数无限制
```

## 生成随机数据

### 随机生成姓名

```sql
select RndName() from xxx limit 100 
```

```sql
create function RndName() returns varchar(32) as 
$$
select concat((array['赵','钱','孙','李','周','吴','郑','王','冯','陈','褚','卫','蒋','沈','韩','杨','朱','秦','尤','许','何','吕','施','张','孔','曹','严','华','金','魏','陶','姜','戚','谢','邹','喻','柏','水','窦','章','云','苏','潘','葛','奚','范','彭','郎','鲁','韦','昌','马','苗','凤','花','方','俞','任','袁','柳','酆','鲍','史','唐','费','廉','岑','薛','雷','贺','倪','汤','滕','殷','罗','毕','郝','邬','安','常','乐','于','时','傅','皮','卞','齐','康','伍','余','元','卜','顾','孟','平','黄','和','穆','萧','尹','姚','邵','舒','汪','祁','毛','禹','狄','米','贝','明','臧','计','伏','成','戴','谈','宋','茅','庞','熊','纪','屈','项','祝','董','杜','阮','蓝','闵','席','季','麻','强','贾','路','娄','危','江','童','颜','郭','梅','盛','林','刁','钟','徐','邱','骆','高','夏','蔡','田','樊','胡','凌','霍','虞','万','支','柯','咎','管','卢','莫','经','房','裘','缪','干','解','应','宗','宣','丁','贲','邓','郁','单','杭','洪','包','诸','左','石','崔','吉','钮','龚','程','嵇','邢','滑','裴','陆','荣','翁','荀','羊','於','惠','甄','加','封','芮','羿','储','靳','汲','邴','糜','松','井','段','富','巫','乌','焦','巴','弓','牧','隗','山','谷','车','侯','宓','蓬','全','郗','班','仰','秋','仲','伊','宫','宁','仇','栾','暴','甘','钭','厉','戎','祖','武','符','刘','詹','束','龙','叶','幸','司','韶','郜','黎','蓟','薄','印','宿','白','怀','蒲','台','从','鄂','索','咸','籍','赖','卓','蔺','屠','蒙','池','乔','阴','胥','能','苍','双','闻','莘','党','翟','谭','贡','劳','逄','姬','申','扶','堵','冉','宰','郦','雍','璩','桑','桂','濮','牛','寿','通','边','扈','燕','冀','郏','浦','尚','农','温','别','庄','晏','柴','瞿','阎','充','慕','连','茹','习','宦','艾','鱼','容','向','古','易','慎','戈','廖','庚','终','暨','居','衡','步','都','耿','满','弘','匡','国','文','寇','广','禄','阙','东','殳','沃','利','蔚','越','夔','隆','师','巩','厍','聂','晁','勾','敖','融','冷','訾','辛','阚','那','简','饶','空','曾','毋','沙','乜','养','鞠','须','丰','巢','关','蒯','相','查','后','红','游','竺','权','逯','盖','益','桓','公','晋','楚','法','汝','鄢','涂','钦','缑','亢','况','有','商','牟','佘','佴','伯','赏','墨','哈','谯','笪','年','爱','阳','佟','琴','言','福'])[trunc(random()*426)+1],(array['烘','匀','彰','飘','宛','戚','褥','邦','煤','缔','慎','窖','爆','笔','柏','懂','孙','娃','潘','洛','阁','捆','怪','捶','变','轴','尤','貌','淮','哺','亏','剐','晤','疗','浅','痛','订','嘲','挥','耙','围','纸','钢','今','渡','冉','愤','绣','公','萧','调','尝','老','充','汝','你','炬','义','蓬','诫','搐','托','絮','账','枪','凑','囊','镶','检','波','承','商','兜','毕','踞','眼','贩','韵','伙','邮','挚','劳','赁','弘','乾','甭','晚','殴','蜗','通','颇','抛','樟','显','剧','唯','顽','侩','爸','瞬','锹','前','呜','施','踊','锅','湍','虏','刽','桑','浮','立','噎','朝','刃','请','卷','棘','连','杰','洽','绝','恼','泻','逻','币','毖','沁','秸','进','搞','横','蜒','敷','霓','榨','绞','土','鄂','猾','哮','缝','罐','三','披','分','潍','眩','皖','摩','茧','压','桓','姚','嚏','俩','固','蕉','宜','顺','忽','誊','陛','原','烽','崔','央','荔','铸','敲','饿','骋','龙','础','湃','柯','遥','慢','损','漆','撕','坑','芜','彻','嗓','庙','恫','待','乡','辨','唉','绑','径','魄','磅','起','妓','卢','梆','常','削','勘','切','黄','漾','七','狠','馏','块','阐','并','哥','佩','邑','仑','驳','膜','睦','硒','接','议','甜','厕','碑','深','阑','岔','虚','豪','驰','慈','莲','高','焉','赡','圣','巷','煌','咎','崖','鲤','蔚','澈','详','逞','颅','宙','诊','矩','壕','信','簧','粘','稼','秦','导','娇','震','帧','滴','阂','拦','设','重','妮','旁','腆','勉','奎','年','帮','榔','远','雪','盟','虫','忿','德','步','饲','柜','亡','敛','缄','迢','泪','神','誓','每','蛤','唐','翠','缅','占','它','割','鞭','饯','匙','蔗','漂','品','穿','咆','奠','亩','娶','西','本','误','噬','登','惦','镁','虎','魁','肆','池','狄','蹄','人','疥','结','讣','畔','廉','釜','操','孵','茸','愁','宝','臻','嫁','似','姨','墅','豺','垦','暇','瀑','涡','橇','乳','帖','沧','墩','斋','辛','配','彝','芒','内','槛','怨','抱','墓','燃','派','牙','向','挟','洋','邓','吠','诺','阅','乐','项','旬','羌','令','成','断','仪','息','甥','藏','拼','想','纷','刑','坝','腹','城','诀','周','帚','碰','舌','陋','祭','残','卤','碳','绚','赖','锁','愧','孩','终','铝','迎','蕊','氖','砸','身','稀','郝','叫','砰','域','肌','收','君','啡','缸','窟','揪','粪','小','讲','哭','蛀','跟','量','逗','薪','背','民','孺','营','版','蚂','折','摧','选','而','贫','惰','蔷','街','肚','纪','陈','桐','酗','渴','辜','滔','恍','吧','喂','抿','去','题','岩','蹭','谤','痉','懈','搀','官','佛','扒','舜','榆','螺','草','怯','睬','昆','栽','由','乞','五','硝','敌','皆','寸','捍','孟','图','汤','篷','弊','吱','录','坷','仓','刹','腰','搂','数','错','堤','硼','捞','播','苟','样','嘻','批','障','顶','论','窃','粳','与','丛','帆','句','裳','翼','炊','谦','狗','票','指','讥','雇','稽','壤','侥','颈','宰','僧','呀','环','忠','志','辙','亮','康','鹏','怕','绵','未','这','豢','讼','死','性','骏','绿','幸','诸','劲','替','匪','么','恐','写','卡','怖','哇','靴','署','褪','斩','涎','懦','段','琵','阵','番','河','廖','必','伐','换','雾','衫','盖','藉','廷','厌','霍','袋','香','短','凡','厩','珍','浆','秆','酒','孔','疆','淀','质','些','祁','权','胀','门','痒','练','宅','抖','盅','味','竖','蓄','打','郊','埠','胡','困','溺','饵','页','弛','腔','盛','亿','臣','技','颁','禹','屋','聪','黔','赏','狱','决','酬','辕','澎','傣','缨','愿','健','记','谁','仇','赛','翻','膀','掂','锐','悲','宫','姐','路','莹','胁','骇','危','鹅','惊','童','氮','拨','胰','梯','豹','惧','古','鬼','锭','除','钠','嵌','圾','第','侨','忱','秘','敝','祥','蛆','撼','匝','箍','奇','勤','蛛','驹','搅','媚','蘸','胯','工','瞻','赋','战','晴','兴','烤','伊','敦','线','驾','缎','率','役','徊','文','飞','傻','影','尸','饺','衬','马','富','歼','嘿','冶','寻','策','丽','鹊','斡','凌','学','幅','帽','刀','贵','旦','置','隧','蚊','瞳','磁','将','谍','沙','轿','挡','机','镐','摄','慕','链','延','屏','糯','云','泡','需','燕','命','研','撵','虾','羔','家','经','苯','遣','狡','祟','恰','矣','急','严','泅','弄','著','渣','娩','握','堪','景','治','郁','扦','芥','俯','史','曰','妙','盐','拄','涤','傲','硫','幌','岛','仗','监','艺','薄','咳','恤','坤','吸','隅','勒','韩','钎','盘','荒','献','况','勇','天','汇','在','形','粹','巴','痢','亥','燎','许','宠','痊','据','鸟','恃','拌','瓦','储','碧','攀','陪','猩','括','钵','冒','留','吩','缚','巩','钮','即','溪','六','汐','侠','蹋','仍','寞','碟','咐','葫','删','递','溢','扯','包','洒','眨','招','川','全','单','慑','夕','佃','楔','讽','刨','绷','逼','尺','渗','昼','霉','扶','鉴','附','颗','牵','谱','跨','雷','涸','赠','见','库','丢','玻','夹','墟','蝇','骡','定','澜','犁','杯','赶','委','蒙','啦','锨','钳','却','己','卞','凤','懒','抗','馁','挞','忧','蔬','袱','峦','采','净','榜','辩','烦','茂','锡','愉','贪','釉','闻','佣','棺','欣','浚','尿','膳','耗','沸','妻','哎','因','糖','象','痈','醛','懊','吓','坟','端','渤','溅','锤','膏','驭','逊','躲','仁','混','嘉','掸','篓','亚','嘱','皇','榷','枣','军','脸','盗','佳','案','班','尔','冤','脐','胜','憋','肄','边','谐','萍','境','拍','于','守','截','鸥','父','烩','肛','摊','敖','栅','音','犯','咒','从','疾','秉','盾','吊','籍','让','酮','予','缆','慰','慨','简','骸','恩','硬','嚣','范','燥','恳'])[floor(random()*998)+1],(array['烘','匀','彰','飘','宛','戚','褥','邦','煤','缔','慎','窖','爆','笔','柏','懂','孙','娃','潘','洛','阁','捆','怪','捶','变','轴','尤','貌','淮','哺','亏','剐','晤','疗','浅','痛','订','嘲','挥','耙','围','纸','钢','今','渡','冉','愤','绣','公','萧','调','尝','老','充','汝','你','炬','义','蓬','诫','搐','托','絮','账','枪','凑','囊','镶','检','波','承','商','兜','毕','踞','眼','贩','韵','伙','邮','挚','劳','赁','弘','乾','甭','晚','殴','蜗','通','颇','抛','樟','显','剧','唯','顽','侩','爸','瞬','锹','前','呜','施','踊','锅','湍','虏','刽','桑','浮','立','噎','朝','刃','请','卷','棘','连','杰','洽','绝','恼','泻','逻','币','毖','沁','秸','进','搞','横','蜒','敷','霓','榨','绞','土','鄂','猾','哮','缝','罐','三','披','分','潍','眩','皖','摩','茧','压','桓','姚','嚏','俩','固','蕉','宜','顺','忽','誊','陛','原','烽','崔','央','荔','铸','敲','饿','骋','龙','础','湃','柯','遥','慢','损','漆','撕','坑','芜','彻','嗓','庙','恫','待','乡','辨','唉','绑','径','魄','磅','起','妓','卢','梆','常','削','勘','切','黄','漾','七','狠','馏','块','阐','并','哥','佩','邑','仑','驳','膜','睦','硒','接','议','甜','厕','碑','深','阑','岔','虚','豪','驰','慈','莲','高','焉','赡','圣','巷','煌','咎','崖','鲤','蔚','澈','详','逞','颅','宙','诊','矩','壕','信','簧','粘','稼','秦','导','娇','震','帧','滴','阂','拦','设','重','妮','旁','腆','勉','奎','年','帮','榔','远','雪','盟','虫','忿','德','步','饲','柜','亡','敛','缄','迢','泪','神','誓','每','蛤','唐','翠','缅','占','它','割','鞭','饯','匙','蔗','漂','品','穿','咆','奠','亩','娶','西','本','误','噬','登','惦','镁','虎','魁','肆','池','狄','蹄','人','疥','结','讣','畔','廉','釜','操','孵','茸','愁','宝','臻','嫁','似','姨','墅','豺','垦','暇','瀑','涡','橇','乳','帖','沧','墩','斋','辛','配','彝','芒','内','槛','怨','抱','墓','燃','派','牙','向','挟','洋','邓','吠','诺','阅','乐','项','旬','羌','令','成','断','仪','息','甥','藏','拼','想','纷','刑','坝','腹','城','诀','周','帚','碰','舌','陋','祭','残','卤','碳','绚','赖','锁','愧','孩','终','铝','迎','蕊','氖','砸','身','稀','郝','叫','砰','域','肌','收','君','啡','缸','窟','揪','粪','小','讲','哭','蛀','跟','量','逗','薪','背','民','孺','营','版','蚂','折','摧','选','而','贫','惰','蔷','街','肚','纪','陈','桐','酗','渴','辜','滔','恍','吧','喂','抿','去','题','岩','蹭','谤','痉','懈','搀','官','佛','扒','舜','榆','螺','草','怯','睬','昆','栽','由','乞','五','硝','敌','皆','寸','捍','孟','图','汤','篷','弊','吱','录','坷','仓','刹','腰','搂','数','错','堤','硼','捞','播','苟','样','嘻','批','障','顶','论','窃','粳','与','丛','帆','句','裳','翼','炊','谦','狗','票','指','讥','雇','稽','壤','侥','颈','宰','僧','呀','环','忠','志','辙','亮','康','鹏','怕','绵','未','这','豢','讼','死','性','骏','绿','幸','诸','劲','替','匪','么','恐','写','卡','怖','哇','靴','署','褪','斩','涎','懦','段','琵','阵','番','河','廖','必','伐','换','雾','衫','盖','藉','廷','厌','霍','袋','香','短','凡','厩','珍','浆','秆','酒','孔','疆','淀','质','些','祁','权','胀','门','痒','练','宅','抖','盅','味','竖','蓄','打','郊','埠','胡','困','溺','饵','页','弛','腔','盛','亿','臣','技','颁','禹','屋','聪','黔','赏','狱','决','酬','辕','澎','傣','缨','愿','健','记','谁','仇','赛','翻','膀','掂','锐','悲','宫','姐','路','莹','胁','骇','危','鹅','惊','童','氮','拨','胰','梯','豹','惧','古','鬼','锭','除','钠','嵌','圾','第','侨','忱','秘','敝','祥','蛆','撼','匝','箍','奇','勤','蛛','驹','搅','媚','蘸','胯','工','瞻','赋','战','晴','兴','烤','伊','敦','线','驾','缎','率','役','徊','文','飞','傻','影','尸','饺','衬','马','富','歼','嘿','冶','寻','策','丽','鹊','斡','凌','学','幅','帽','刀','贵','旦','置','隧','蚊','瞳','磁','将','谍','沙','轿','挡','机','镐','摄','慕','链','延','屏','糯','云','泡','需','燕','命','研','撵','虾','羔','家','经','苯','遣','狡','祟','恰','矣','急','严','泅','弄','著','渣','娩','握','堪','景','治','郁','扦','芥','俯','史','曰','妙','盐','拄','涤','傲','硫','幌','岛','仗','监','艺','薄','咳','恤','坤','吸','隅','勒','韩','钎','盘','荒','献','况','勇','天','汇','在','形','粹','巴','痢','亥','燎','许','宠','痊','据','鸟','恃','拌','瓦','储','碧','攀','陪','猩','括','钵','冒','留','吩','缚','巩','钮','即','溪','六','汐','侠','蹋','仍','寞','碟','咐','葫','删','递','溢','扯','包','洒','眨','招','川','全','单','慑','夕','佃','楔','讽','刨','绷','逼','尺','渗','昼','霉','扶','鉴','附','颗','牵','谱','跨','雷','涸','赠','见','库','丢','玻','夹','墟','蝇','骡','定','澜','犁','杯','赶','委','蒙','啦','锨','钳','却','己','卞','凤','懒','抗','馁','挞','忧','蔬','袱','峦','采','净','榜','辩','烦','茂','锡','愉','贪','釉','闻','佣','棺','欣','浚','尿','膳','耗','沸','妻','哎','因','糖','象','痈','醛','懊','吓','坟','端','渤','溅','锤','膏','驭','逊','躲','仁','混','嘉','掸','篓','亚','嘱','皇','榷','枣','军','脸','盗','佳','案','班','尔','冤','脐','胜','憋','肄','边','谐','萍','境','拍','于','守','截','鸥','父','烩','肛','摊','敖','栅','音','犯','咒','从','疾','秉','盾','吊','籍','让','酮','予','缆','慰','慨','简','骸','恩','硬','嚣','范','燥','恳'])[floor(random()*2000)+1]);
$$ language sql
```

### 随机生成身份证号码

```sql
select RndIdcard() 
```

```sql
create or replace function RndIdcard() returns varchar(18) as 
$$
select concat(floor(random()*(500000-777799)+999999),to_char(now()::timestamp - random() * interval '10 year','YYYYMMDD'),floor(random()*(10-99)+99),floor(random()*2),floor(random()*9))
$$ language sql

```

### 随机生成手机号

```sql
create or replace function RndPhone() returns varchar(18) as 
$$
select to_char(floor(random()*(13000000000-13999999999)+13999999999),'00000000000')
$$ language sql

```

### 随机生成日期时间

```pgsql
create or replace function RndDatetime() returns TIMESTAMP as 
$$
select now()::timestamp - random() * interval '3 month'
$$ language sql

```

### 随机生成设备id

```pgsql
create or replace function RndDevid() returns varchar(32) as 
$$
select concat('DEV-',floor(random()*(300000-700000)+700000),floor(random()*(1000000-7000000)+7000000))
$$ language sql
```

### 建立列存表

```pgsql
create table demo_person_pass (id serial8,name varchar(32),idcard varchar(32),phone varchar(16),equipment_id varchar(32),pass_time timestamptz) with(appendonly=true,orientation=column) distributed by (id) ;
```

### 写入测试数据

```pgsql
insert into demo_person_pass(name,idcard,phone,equipment_id,pass_time) SELECT
	RndName ( ) NAME,
	RndIdcard ( ) idcard,
	RndPhone ( ) phone,
	RndDevid() equipment_id,
	RndDatetime ( ) pass_time 
FROM
	user_log 
	LIMIT 1000
```

## 创建用户

```pgsql
create user zyzs with password 'pwd';
grant all privileges on database zyzsdag to zyzs;
\c dbname
grant all privileges on all tables in schema public to zyzs;
```