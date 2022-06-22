# gp/mdb集群内存配置
> 由于postgres为多进程模式，在设置好内存各参数的情况下还必须结合服务端连接池来降低资源占用压力。
## POSTGRES 层面内存设置
+ shared_buffers 一般为物理内存的25%
+ Effective_cache_size 用于优化查询器内存估算使用，一般为内存的1/2到3/4
+ work_mem 主要用于结果排序，单个连接最少使用一份work_mem的内存，最大连接数设置需要参考此项，注意并非一个连接只占用一份work_mem！！一个复杂查询可能会占用多份的work_mem的内存，默认32MB
+ maintenance_work_mem 用于维护使用的最大内存量Vacuum、Create Index和Alter Table Add Foreign Key 默认64MB，由于通常正常运行的数据库中不会有大量并发的此类操作，可以设置的较大一些，提高清理和创建索引外键的速度。
+ wal_buffers 在PostgreSQL 12版中，默认值为-1，也就是选择等于shared_buffers的1/32 。如果自动的选择太大或太小可以手工设置该值。一般考虑设置为16MB。
+ temp_buffers 设置每个数据库会话使用的临时缓冲区的最大数目。此本地缓冲区只用于访问临时表。临时缓冲区是在某个连接会话的服务进程中分配的，属于本地内存。临时缓冲区的大小也是按数据块大小分配的，默认8MB。

## GP/MDB层面内存设置
+ gp_vmem_protect_limit 物理内存 × 0.9 ÷ 主机上分片数量（激进点就除主分片数量）单位MB

```
# 以单个节点256GB内存分片数12（6主+6从）为例的最佳配置
# 可分配内存以实际内存的90%（给系统预留一些）256*0.9=230GB
# 单个分片最大可用内存为230/12=19GB 
# 使用https://pgtune.leopard.in.ua/ 工具计算
# 核心数48/12=4
# DB Version: 13
# OS Type: linux
# DB Type: mixed
# Total Memory (RAM): 19 GB
# CPUs num: 4
# Data Storage: hdd
#主节点连接数
max_connections = 100
shared_buffers = 4864MB
effective_cache_size = 14592MB
maintenance_work_mem = 1216MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 12451kB
min_wal_size = 1GB
max_wal_size = 4GB
#max_worker_processes = 4
#max_parallel_workers_per_gather = 2
#max_parallel_workers = 4
#max_parallel_maintenance_workers = 2
gp_vmem_protect_limit=11831
statement_mem=200MB
#数据节点连接数gpconfig -c max_connections -m 100 -v 500
max_connections=500
max_prepared_transactions =100
```

```
# 单节点500GB内存分片数8（4主4从）为例
# 500*0.9=450GB
# 单个分片最大可用内存=450/8=56GB
# 核心数56/8=7
# DB Version: 13
# OS Type: linux
# DB Type: mixed
# Total Memory (RAM): 56 GB
# CPUs num: 7
# Data Storage: hdd

max_connections = 150
shared_buffers = 14GB
effective_cache_size = 42GB
maintenance_work_mem = 2GB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 18350kB
min_wal_size = 1GB
max_wal_size = 4GB
max_worker_processes = 7
max_parallel_workers_per_gather = 4
max_parallel_workers = 7
max_parallel_maintenance_workers = 4
gp_vmem_protect_limit=50864
statement_mem=400MB
#数据节点连接数gpconfig -c max_connections -m 150 -v 750
max_connections=750
max_prepared_transactions =150
```


> 上述参数需要通过gpconfig -c work_mem -m '18350kB' -v '18350kB' 这样的形式在管理节点统一执行，这样才能自动分发到数据节点去

## pgbouncer配置
+ 准备日志和pid目录
```bash
mkdir /var/log/pgbouncer
chown mxadmin:mxadmin -R /var/log/pgbouncer
mkdir /var/run/pgbouncer
chown mxadmin:mxadmin -R /var/run/pgbouncer

```
+ 创建连接池配置文件
```
#/etc/matrixdb/pguserlist.txt 这里的内容来自与psql -c "select usename,passwd from pg_shadow;",查出的结果给值都加上双引号放进来即可。
"mxadmin"	"md5d802636a5d23d0fd440e212213e94d92"
```

#/etc/matrixdb/pgbouncer.ini
[databases]
lhcz1 = host=127.0.0.1 port=5433
lhcz = host=127.0.0.1 port=5434
[pgbouncer]
logfile = /var/log/pgbouncer/pgbouncer.log
log_connections=0
log_disconnections=0
log_stats=0
pidfile = /var/run/pgbouncer/pgbouncer.pid
listen_addr = 0.0.0.0
listen_port = 5432
auth_type = hba
auth_file = /etc/matrixdb/pguserlist.txt
auth_hba_file = /etc/matrixdb/pg_hba.conf
admin_users = mxadmin,hjdev
stats_users = stats
pool_mode = transaction
server_reset_query = DISCARD ALL
max_client_conn = 800
default_pool_size = 100
ignore_startup_parameters = extra_float_digits,geqo
```
```
#/etc/matrixdb/pg_hba.conf
local	all	all	trust
host	all	all	0.0.0.0/0	md5
```

+ 使连接池保持自启动
```
#新增文件/etc/matrixdb/supervisor.pgbouncer.conf
[program:pgbouncer]
  process_name=pgbouncer
  command=/usr/local/matrixdb/bin/pgbouncer /etc/matrixdb/pgbouncer.ini -u mxadmin
  directory=/etc/matrixdb
  autostart=true
  autorestart=unexpected
  exitcodes=0
  process_num=1
  numprocs_start=0
  stopsignal=TERM
  stopwaitsecs=5
  stopasgroup=true
  killasgroup=true
  stdout_logfile=%(ENV_MXLOGDIR)s/pgbouncer.log
  stdout_logfile_maxbytes=50MB
  stdout_logfile_backups=10
  redirect_stderr=true
```
```
#修改/etc/matrixdb/supervisor.conf 在最后include段的files配置项的值后追加下面内容（注意和原有内容之间有个空格）
 supervisor.pgbouncer.conf
```
+ 查看与重启
```bash
/usr/local/matrixdb/bin/supervisord -c /etc/matrixdb/supervisor.conf ctl -s http://localhost:4617 -u matrixdb -P changeme status
/usr/local/matrixdb/bin/supervisord -c /etc/matrixdb/supervisor.conf ctl -s http://localhost:4617 -u matrixdb -P changeme restart pgbouncer
```
## 程序配置
#datax或者有jdbcurl可以配置的程序在参数里增加下面这个设置项,端口使用连接池的6543端口
prepareThreshold=0
#其他类型程序比如ftp2gp类型的单纯修改端口为6543即可。


## 排查尚未通过连接池链接的应用
#命令行的psql无需修改，依然通过原来的5432链接

```sql
select client_addr,client_port,usename,application_name,state,pid from pg_stat_activity ;
#可以查询出客户端链接的ip和端口比如ip尾号123端口1234可以ssh链接过去netstat -anop|grep 1234找到对应的进程号，再通过ps -aux|grep 进程号找到使哪个程序，修改其配置重启即可。
```
