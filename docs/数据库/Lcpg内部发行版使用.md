# Lcpg内部发行版使用

## 初始化
```bash
mkdir /data/pg14
tar zxvf pg14py.tgz -C /data/pg14
cd /data/pg14
rpm -ivh py/*.rpm
pip3 install --no-index -f py/pip fabric simplejson sh

```
# dev

## initenv.sh
```bash
su postgres -c "initdb --no-locale -E UTF_8 --data-checksums -D data"
sed -i 's/port/#port/g' data/postgresql.conf
sed -i 's/listen_addresses/#listen_addresses/g' data/postgresql.conf
echo 'port=5433' >> data/postgresql.conf
echo "listen_addresses='*'" >> data/postgresql.conf
echo "max_wal_senders=5" >> data/postgresql.conf
echo "wal_level=hot_standby" >> data/postgresql.conf
echo "logging_collector=on" >> data/postgresql.conf
echo "host replication all mdw1 trust" >> data/pg_hba.conf
#备节点
mkdir /data/pg14/data -p
chown postgres:postgres -R /data/pg14
su postgres -c "pg_basebackup -h mdw -U postgres -F p -P -R -D /data/pg14/data -p 5433 -l /data/pg14/standbylog"
echo "primary_conninfo='application_name=mdw1 user=postgres host=mdw port=5433 sslmode=disable sslcompression=1 target_session_attrs=any'" >> data/postgresql.conf
su postgres -c "touch data/standby.signal"
chmod 700 -R data
su postgres -c "pg_ctl start -D data"
```



## centos7构建

```bash
yum install --downloadonly --downloaddir ./ citus110_14 postgresql14-contrib nano pgbouncer wal2json_14 topn_14 pg_cron_14 pg_repack_14 timescaledb_14 sequential_uuids_14 postgresql_faker_14 pglogical_14 pgaudit16_14 pg_top_14 pg-auto-failover16_14 pg_auto_failover_14 mysql_fdw_14 mongo_fdw_14 hll_14 hdfs_fdw_14 pgpool-II pgpoolAdmin pgpool-II-pg14-extensions pgadmin4-web barman barman-cli python36-sh createrepo ed

createrepo .

cat <<EOF > /etc/yum.repos.d/citus.repo
[citusrep]
name=citusrep
baseurl=http://localhost:6789
enabled=1
gpgcheck=0
EOF

yum install --disablerepo=* --enablerepo=citusrep 
```

+ 安装
```bash
#解压
nohup python2 -m SimpleHTTPServer 6789 &
cat <<EOF > /etc/yum.repos.d/citus.repo
[citusrep]
name=citusrep
baseurl=http://localhost:6789
enabled=1
gpgcheck=0
EOF

yum makecache --disablerepo=* --enablerepo=citusrep

yum install --disablerepo=* --enablerepo=citusrep -y citus110_14 postgresql14-contrib nano pgbouncer wal2json_14 topn_14 pg_cron_14 pg_repack_14 timescaledb_14 sequential_uuids_14 postgresql_faker_14 pglogical_14 pgaudit16_14 pg_top_14 pg_auto_failover_14 mysql_fdw_14 mongo_fdw_14 hll_14 hdfs_fdw_14 pgpool-II pgpoolAdmin ed pgpool-II-pg14-extensions pgadmin4-web barman barman-cli python36-sh

echo "export PATH=$PATH:/usr/pgsql-14/bin" >> /var/lib/pgsql/.bash_profile
echo "export PATH=$PATH:/usr/pgsql-14/bin" >> ~/.bash_profile
mkdir /data/pgdata
chown postgres:postgres -R /data/pgdata
su - postgres


```
## pg_auto_failover 版本高可用
+ 环境准备
  + 所有节点配置ntp
  + 所有节点关闭selinux并重启
+ 初始化
```bash
#以下命令将运行一个监控节点和一主一从的集群，客户端访问时候通过5001和5002访问其中一个是可读写另一个是只读。
#root执行
setsebool 0

#postgres执行
export PGDATA=./monitor
export PGPORT=5000
pg_autoctl create monitor --pgdata /data/pgdata/monitor --pgport 5001 --hostname mdw --auth trust --ssl-self-signed

export PGDATA=./node_1
export PGPORT=5001
pg_autoctl create postgres --hostname localhost --auth trust --ssl-self-signed --monitor 'postgres://autoctl_node@localhost:5000/pg_auto_failover?sslmode=require' 

export PGDATA=./node_2
export PGPORT=5002
pg_autoctl create postgres --hostname localhost --auth trust --ssl-self-signed --monitor 'postgres://autoctl_node@localhost:5000/pg_auto_failover?sslmode=require' 
#root执行
pg_autoctl -q show systemd --pgdata /data/pgdata/monitor > /usr/lib/systemd/system/pgmonitor.service
pg_autoctl -q show systemd --pgdata /data/pgdata/node_1 > /usr/lib/systemd/system/pgnode1.service
pg_autoctl -q show systemd --pgdata /data/pgdata/node_2 > /usr/lib/systemd/system/pgnode2.service
systemctl daemon-reload
systemctl enable pgmonitor
systemctl enable pgnode1
systemctl enable pgnode2
systemctl start pgmonitor
systemctl start pgnode1
systemctl start pgnode2
```

## pgpoolII版本高可用
+ 环境准备
  + 所有节点配置ntp
  + 所有节点关闭selinux并重启
+ 初始化
```bash
#root执行
setsebool 0
#postgres执行
initdb -E utf-8 -D node1
#修改pg_hba让从节点可以免密访问
echo "host replication all 0.0.0.0/0 trust" >> node1/pg_hba.conf
echo "host all all 0.0.0.0/0 md5"  >> node1/pg_hba.conf
#修改监听地址
cat <<EOF >> node1/postgresql.conf
listen_addresses = '*'
port = 5433
max_wal_senders=5 # 大于备库数量
wal_level=hot_standby
EOF

#在其他节点初始化备库
pg_basebackup -h mdw -U postgres -F p -P -R -D /data/pgdata/node2 -p 5433
echo "primary_conninfo='application_name=mdw1 user=postgres host=mdw port=11002 sslmode=disable sslcompression=1 target_session_attrs=any'" >> node2/postgresql.conf
touch node2/standby.signal
pg_ctl start -D node2

#主节点postgres用户查看状态
psql -p 5433 -c 'select * from pg_stat_replication;'

#配置pgpool负载均衡
mkdir {etc,run,cache}

```