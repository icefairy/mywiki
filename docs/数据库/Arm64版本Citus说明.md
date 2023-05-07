# 初始化数据库

```bash
mkdir /data/pgdata -p
chmod 777 -R /data/pgdata
docker run -it --rm -v /data/pgdata:/var/lib/postgresql -u postgres mycitusext:1.0 initdb -E utf-8
#修改/data/pgdata/data/pg_hba.conf中增加相应的链接权限设置例如host all all 0.0.0.0/0 md5来允许外部ip链接
#修改/data/pgdata/data/postgresql.conf中相应的参数，例如连接数以及日志配置，加载的插件等(可以参考下面的样例)

```

postgresql.conf配置样例

```
max_connections = 200
logging_collector = on
log_filename = 'postgresql-%W.log'
log_rotation_size = 1GB
log_timezone='PRC'
timezone = 'PRC'
cron.timezone='PRC'
cron.database_name = 'lhcz'
shared_preload_libraries = 'citus,pg_cron'


```


```bash
#正式启动数据库
docker run -itd --name mypg --net host --restart=always -v /data/pgdata:/var/lib/postgresql -u postgres mycitusext:1.0 postgres
#设置用户
docker exec -it mypg bash
createuser -d -P --replication hjdev
输入两次密码
#创建数据库并设置所有者
createdb -E utf-8 -O hjdev lhcz
#创建定时任务插件
psql lhcz
create extension pg_cron;
create extension citus;
create extension mysql_fdw;
\dx 查看生效的插件应该显示为
citus          | 11.1-1  | pg_catalog | Citus distributed database
citus_columnar | 11.1-1  | pg_catalog | Citus Columnar extension
mysql_fdw      | 1.1     | public     | Foreign data wrapper for querying a MySQL server
pg_cron        | 1.4-1   | public     | Job scheduler for PostgreSQL
plpgsql        | 1.0     | pg_catalog | PL/pgSQL procedural language




```
