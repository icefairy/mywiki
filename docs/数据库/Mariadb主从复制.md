## 创建容器

```bash
docker network create mdb
rootpwd=`openssl rand -base64 16`
echo $rootpwd
#K0XwkqGwPH+2g4A8x/OueQ==
#如果是跨主机部署可以直接指定--net host即可使用主机网络无需映射端口
docker run -itd --name pridb -e MARIADB_ROOT_PASSWORD="K0XwkqGwPH+2g4A8x/OueQ==" --net mdb --hostname pridb -p 3306:3306 mariadb:10.5
docker run -itd --name secdb -e MARIADB_ROOT_PASSWORD="K0XwkqGwPH+2g4A8x/OueQ==" --net mdb --hostname secdb -p 3307:3306 mariadb:10.5
```

## 设置主节点

```bash
docker exec -it pridb bash
cd /etc/mysql/mariadb.conf.d
#可以在下面配置文件中自定义保留logbin的日期，保留的长了会占用更多磁盘空间，短了会存在从节点掉线超过N天就无法再同步的情况，需要自行选择
cat > pri.cnf
[mysqld]
server-id = 1
log-bin = mysql-bin
expire_logs_days=3
Ctrl+D
exit
docker restart pridb
docker exec -it pridb bash
mysql -uroot -p
#查看主节点状态
show master status;
#创建从节点复制账户
CREATE USER 'sec'@'%' IDENTIFIED BY 'K0XwkqGwPH';
GRANT REPLICATION SLAVE ON *.* TO 'sec'@'%';


```

## 配置从节点

```bash
docker exec -it secdb bash
cd /etc/mysql/mariadb.conf.d
cat > sec.cnf
[mysqld]
server-id=2
Ctrl+D
exit
docker restart secdb
docker exec -it pridb bash
mysql -uroot -p
#创建账户
CREATE USER 'sec'@'%' IDENTIFIED BY 'K0XwkqGwPH';
GRANT REPLICATION SLAVE ON *.* TO 'sec'@'%';
#下面的值通过在主节点中的show master status获得
CHANGE MASTER TO MASTER_HOST='pridb', MASTER_USER='sec', MASTER_PASSWORD='K0XwkqGwPH', MASTER_LOG_FILE='mysqld-bin.000001', MASTER_LOG_POS=646;
start slave;
show slave status;
```

## 测试

```sql
--在主节点执行
create database testdb;
use testdb;
create table mytest(id int,txt varchar(255));
insert into mytest values(1,'111');
--在从节点执行
show databases;
use testdb;
select * from mytest;
--即可看到testdb中的mytest表中的数据

```