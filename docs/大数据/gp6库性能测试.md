#gp#

> 本测试直接基于实际业务使用场景，通过jdbc的批量insert方式写入数据对数据库的写入、读取性能进行相应的测试
>

## 测试数据表

```sql
CREATE TABLE "public"."muser" (
"id" int4 DEFAULT nextval('muser_id_seq'::regclass) NOT NULL,
"col1" varchar(255) COLLATE "default",
"user" varchar(255) COLLATE "default",
"pass" varchar(255) COLLATE "default",
"created" timestamp(0)
)
WITH (OIDS=FALSE)
```

## 测试代码

```kotlin
class gptestTest {
    val log=LogFactory.get()
    @Test
    fun insertData1(){
        val db= Db.use()
        for (num in arrayOf(1,10,100)){
            val lst=ArrayList<Entity>()
            val ts= measureTimeMillis {
                for (i in 0 until 10000*num){
                    val it=Entity.create("muser")
                    it.set("col1","user${i+1}")
                    it.set("user","testuser${i+1}")
                    it.set("pass","tp${i}_123456")
                    it.set("created", Date())
                    lst.add(it)
                    if (i % 1000 ==0 ){
                        db.insert(lst)
                        lst.clear()
                    }
                }
            }
            log.info("非事务批量方式（insert into values(),(),()）写入${10000*num}条数据耗时:$ts ms")
        }

    }
}
```

## 当前方案（mysql）写入测试

> 非事务批量方式（insert into values(),(),()）
>

* 1*10000条数据写入总耗时9244ms,平均每秒1111条
* 10*10000条数据写入总耗时95018ms,平均每秒1052条
* 100*10000条数据写入总耗时989530ms，平均每秒1010条
* 测试过程中网络带宽最高300KB/s

> 事务批量方式
>

* 1*10000条数据写入总耗时8959ms,平均每秒1111条
* 10*10000条数据写入总耗时90899ms,平均每秒1098条
* 100*10000条数据写入总耗时989530ms，平均每秒1010条
* 测试过程中网络带宽最高300KB/s

## gp库默认设置测试

> 非事务批量方式（insert into values(),(),()）
>

* 1*10000条数据写入总耗时12473ms,平均每秒883条
* 10*10000条数据写入总耗时111765ms,平均每秒892条
* 20*10000条数据写入总耗时223120ms，平均每秒897条
* 测试过程中网络带宽最高120KB/s

> 事务批量方式（insert into values(),(),()）
>

* 1*10000条数据写入总耗时9989ms,平均每秒1000条
* 10*10000条数据写入总耗时104827ms,平均每秒952条
* 20*10000条数据写入总耗时211133ms，平均每秒948条
* 测试过程中网络带宽最高120KB/s

## gp库调优后测试

```bash
#调优方案
gpconfig -c gp_enable_global_deadlock_detector -v on
#下面这个对写入影响不大，建议保留
#gpconfig -c optimizer -v off
gpconfig -c log_statement -v none
gpconfig -c wal_writer_delay -v '10ms'
gpconfig -c bgwriter_delay -v '10ms' --skipvalidation
gpconfig -c commit_siblings -v 16 --skipvalidation
gpconfig -c checkpoint_completion_target -v 0.9
gpconfig -c bgwriter_lru_maxpages -v 1000 --skipvalidation
gpconfig -c fsync -v off --skipvalidation
gpconfig -c full_page_writes -v off --skipvalidation
gpconfig -c wal_buffers -v '32MB' -m '256MB'
gpconfig -c shared_buffers -v '512MB' -m '4GB'
gpconfig -c resource_scheduler -v off

#重载修改的配置
gpstop -u 
#检查修改是否生效
gpconfig -s gp_enable_global_deadlock_detector && gpconfig -s log_statement
```

> 非事务批量方式（insert into values(),(),()）
>

* 1*10000条数据写入总耗时9642ms,平均每秒1000条
* 10*10000条数据写入总耗时101516ms,平均每秒980条
* 20*10000条数据写入总耗时201291ms，平均每秒995条
* 50*10000条数据写入总耗时525474ms，平均每秒952条
* 100*10000条数据写入总耗时1064118ms，平均每秒955条
* 测试过程中网络带宽最高150KB/s

> 事务批量方式（insert into values(),(),()）
>

* 1*10000条数据写入总耗时9497ms,平均每秒1111条
* 10*10000条数据写入总耗时100246ms,平均每秒1000条
* 20*10000条数据写入总耗时200010ms，平均每秒1000条
* 50*10000条数据写入总耗时519329ms，平均每秒963条
* 100*10000条数据写入总耗时1047257ms，平均每秒955条
* 测试过程中网络带宽最高150KB/s

## 单节点postgresql默认配置测试

> docker run -itd --name psql96 -v /data/psqldata:/var/lib/postgresql/data -p 15432:5432 -e POSTGRES_PASSWORD=Bzpg123456 hdfs4:5000/postgresql:9.6
> 非事务批量方式（insert into values(),(),()）
>

* 1*10000条数据写入总耗时1334ms,平均每秒10000条
* 10*10000条数据写入总耗时10575ms,平均每秒9091条
* 100*10000条数据写入总耗时103150ms，平均9709条
* 测试过程中网络带宽最高1.2MB/s

> 事务批量方式（insert into values(),(),()）
>

* 1*10000条数据写入总耗时964ms,平均每秒10000条
* 10*10000条数据写入总耗时10362ms,平均每秒10000条
* 100*10000条数据写入总耗时104843ms，平均每秒10000条
* 测试过程中网络带宽最高1.3MB/s

## 写入重复数据

直接批量写入不重复的:10508ms
通过rule写入1万条数据其中包括10条不重复的，写入耗时19473ms

## 测试结论

* 在3个数据节点的集群环境下，4个机器分别运行写入测试程序往同一个数据表中写入数据，得出的平均写入速度下降和单个写入程序的平均速度下降是否有限总体在10%~20%之间，并发写入性能在1000条每秒左右，4份程序并发写入能达到4000条每秒，对业务系统来说性能十分充足
* greenplum分布式数据库写入性能可以满足业务需要
* 基于datax的gpdb插件使用copy模式批量写入数据9325万耗时2.14小时 平均每秒钟写入12189条
* 导入9325万车辆通行数据后直接通过通行时间加车牌号查询返回100条耗时4.5秒左右，车号通行时间字段添加索引后（9000万数据添加索引耗时3分钟）同样语句修改不同的查询范围以及筛选条件查询耗时0.01~0.15秒左右，查询性能满足业务需求