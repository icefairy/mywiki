```
gpconfig -c gp_enable_global_deadlock_detector -v on
gpconfig -c log_statement -v none
gpconfig -c wal_writer_delay -v '10ms'
gpconfig -c checkpoint_completion_target -v 0.9
gpconfig -c shared_buffers -v '16GB' -m '512MB'
gpconfig -c work_mem -v '16MB' -m '16MB'
gpconfig -c wal_buffers -v '10MB' -m '10MB'
gpconfig -c statement_mem -v '2GB' -m '2GB'
gpconfig -c max_statement_mem -v '4GB' -m '4GB'
gpconfig -c statement_timeout -m 10min -v 10min
#设置时区
gpconfig -c TimeZone -m 'Asia/Shanghai' -v 'Asia/Shanghai'
#自动分析
gpconfig -c gp_autostats_mode -m on_change -v on_change
gpconfig -c gp_autostats_on_change_threshold -m 100000000 -v 100000000
#设置内存大小，最小内存节点内存=x，此值计算方法为=x*0.75/(主节点数+从节点数)
gpconfig -c gp_vmem_protect_limit -m 6144 -v 6144
#修改最大连接数(-v为segment的官方建议为主节点的5~10倍)
gpconfig -c max_connections -v 4000 -m 2000 && gpconfig -c max_prepared_transactions -v 4000
#重启集群生效（部分配置需要彻底重启）
gpstop -M fast -a && gpstart 
#检查修改是否生效
gpconfig -s gp_enable_global_deadlock_detector && gpconfig -s log_statement
gpconfig -s max_connections && gpconfig -s max_prepared_transactions

#创建数据库
createdb lhcz -E utf-8
#创建用户
psql -d lhcz
create role hjdev password 'thisispass' createdb login;
\q 
#在master节点修改pg_hba.conf文件来赋予用户远程登录的权限
vi /home/gpadmin/master/gpseg-1/pg_hba.conf
#添加如下内容
host all hjdev 0.0.0.0/0 md5
:wq 保存退出
#重新加载配置
gpstop -u
```

## pgbouncer配置

```
[databases]
lhcz = host=localhost port=5432 dbname=lhcz
bzyq = host=localhost port=5432 dbname=bzyq
test = host=localhost port=5432 dbname=test
postgres = host=localhost port=5432 dbname=postgres

[pgbouncer]
listen_port=6543
listen_addr=0.0.0.0
auth_type=md5
auth_file=userlist.txt
logfile=pgbouncer.log
pidfile=pgbouncer.pid
max_db_connections=500
admin_users=hjdev
;next for datax
ignore_startup_parameters = extra_float_digits
;; When server connection is released back to pool:
;;   session      - after client disconnects (default)
;;   transaction  - after transaction finishes ;datax throw error
;;   statement    - after statement finishes ;cannot use transaction
;pool_mode = transaction
```

* transaction模式会造成datax报错，需要在datax的数据库连接字串中增加`prepareThreshold=0`
* 创建用户配置文件userlist.txt(如下)

```
"用户名" "密码明文"
"用户名" "md5+md5值"
```

## 使用pgbouncer

```
#进入连接池管理库
psql -p 6543 -h 127.0.0.1 -U hjdev pgbouncer
#输入密码
show help;#查看帮助
show clients;#查看连接的客户端
```