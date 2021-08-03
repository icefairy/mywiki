#  greenplum集群搭建

> 这是使用docker方式搭建的，仅供参考，实际生产环境建议直接在宿主机安装

## 机器列表

- 123.123.4.96
- 123.123.4.100
- 123.123.4.101
- 123.123.4.102
- 123.123.4.103 主节点独占
- 123.123.4.104
- 123.123.4.105

## docker swarm方式(当前采用)

> docker 版本19.03

- 同步所有机器时钟

```
ntpdate 21.0.2.8
```

- 更新内核到至少4.4

```bash
  rpm -ivh http://ip:9999/kernel-ml-5.1.16-1.el7.elrepo.x86_64.rpm && sed -i "s/saved/0/g" /etc/default/grub && grub2-mkconfig -o /boot/grub2/grub.cfg
  grub2-set-default 'CentOS Linux (5.1.16-1.el7.elrepo.x86_64) 7 (Core)'
  grub2-editenv list
  reboot
```

- 修改容器宿主机 /etc/security/limit.conf中的最大打开文件数量限制

- 系统资源限制设置

```bash
#/etc/security/limits.conf 
* soft nofile 524288
* hard nofile 524288
* soft nproc 131072
* hard nproc 131072
root soft nofile 524288
root hard nofile 524288
root soft nproc 131072
root hard nproc 131072
ulimit -u
```

- 容器宿主机系统内核设置 /etc/sysctl.conf

```
#下面这个值来自# echo $(expr $(getconf _PHYS_PAGES) / 2) 
kernel.shmall=25769461
#下面这个值来自# echo $(expr $(getconf _PHYS_PAGES) / 2 \* $(getconf PAGE_SIZE))
kernel.shmmax=105551712256
kernel.shmmni = 4096
vm.overcommit_memory = 2
vm.overcommit_ratio = 95

net.ipv4.ip_local_port_range = 10000 65535
#下面这个值影响最大连接数
kernel.sem = 50100 128256000 50100 2560
kernel.sysrq = 1
kernel.core_uses_pid = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.msgmni = 2048
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.conf.all.arp_filter = 1
net.core.netdev_max_backlog = 10000
net.core.rmem_max = 2097152
net.core.wmem_max = 2097152
vm.swappiness = 10
vm.zone_reclaim_mode = 0
vm.dirty_expire_centisecs = 500
vm.dirty_writeback_centisecs = 100
vm.dirty_background_ratio = 0
vm.dirty_ratio = 0
vm.dirty_background_bytes = 1610612736
vm.dirty_bytes = 4294967296
```

- 容器宿主机修改数据盘的挂载选项为:`rw,nodev,noatime,nobarrier,inode64`

```
nano /etc/fstab
#如果类型已经是xfs就光修改选项即可，gp官方推荐xfs类型，如果没啥数据且不是xfs的话可以格式化一下
#重新加载分区
mount -o,remount /data
#其中105为当前正在使用的机器不便格式化数据分区，考虑暂时不将其加入集群中
```

- 创建docker swarm集群

```bash
docker swarm init
#worker
docker swarm join --token SWMTKN-1-3t4l2pk38x0v08f81u5kamsd6izx8llun3wy34oq3emvwy2p9x-1j0uk1hytunbpwt5lvzztauu3 123.123.4.104:2377
#manager
docker swarm join --token SWMTKN-1-3t4l2pk38x0v08f81u5kamsd6izx8llun3wy34oq3emvwy2p9x-3y9hzhemm08k70dh4s0f3uw5x 123.123.4.104:2377
```

- 创建网络

```
  docker network create --attachable -d overlay --subnet 192.168.123.0/24 gpnet
```

- 创建主节点和数据节点

```bash
  #主节点103
  mkdir /data/gpmaster && chmod 777 -R /data/gpmaster
  docker run -itd --name mdw --hostname mdw --net gpnet -p 6222:22 -p 5432:5432 -v /data/gpmaster:/home/gpadmin/master --restart=always hdfs4:5000/mygreenplum6:fdwmo
  #数据节点101
  mkdir /data/{gpdata,gpmirror} && chmod 777 -R /data/gpdata && chmod 777 -R /data/gpmirror
  docker run -itd --name sdw4 --hostname sdw4 --net gpnet -v /data/gpmirror:/home/gpadmin/mirror -v /data/gpdata:/home/gpadmin/data --restart=always hdfs4:5000/mygreenplum6:fdwmo
  #数据节点102
  mkdir /data/{gpdata,gpmirror} && chmod 777 -R /data/gpdata && chmod 777 -R /data/gpmirror
  docker run -itd --name sdw2 --hostname sdw2 --net gpnet -v /data/gpmirror:/home/gpadmin/mirror -v /data/gpdata:/home/gpadmin/data --restart=always hdfs4:5000/mygreenplum6:fdwmo
  #数据节点104
  mkdir /data/{gpdata,gpmirror} && chmod 777 -R /data/gpdata && chmod 777 -R /data/gpmirror
  docker run -itd --name sdw3 --hostname sdw3 --net gpnet -v /data/gpmirror:/home/gpadmin/mirror -v /data/gpdata:/home/gpadmin/data --restart=always hdfs4:5000/mygreenplum6:fdwmo
  #数据节点100
  mkdir /data/{gpdata,gpmirror} && chmod 777 -R /data/gpdata && chmod 777 -R /data/gpmirror
  docker run -itd --name sdw1 --hostname sdw1 --net gpnet -v /data/gpmirror:/home/gpadmin/mirror -v /data/gpdata:/home/gpadmin/data --restart=always hdfs4:5000/mygreenplum6:fdwmo
  #备份主节点103
  mkdir /data/gpdata && chmod 777 -R /data/gpdata
  docker run -itd --name mdw1 --hostname mdw1 --net gpnet -v /data/gpstandby:/home/gpadmin/master --restart=always hdfs4:5000/mygreenplum6:fdwmo
  #备份主节点96
  mkdir /data/{gpdata,gpmirror} && chmod 777 -R /data/gpdata && chmod 777 -R /data/gpmirror
  docker run -itd --name sdw5 --hostname sdw5 --net gpnet -v /data/gpmirror:/home/gpadmin/mirror -v /data/gpdata:/home/gpadmin/data --restart=always hdfs4:5000/mygreenplum6:fdwmo
```

- 初始化集群

```bash
  #103主节点执行
  docker exec -it -u gpadmin mdw bash
  cd ~
  source /usr/local/greenplum-db/greenplum_path.sh
  ./artifact/prepare.sh -s 4 -n 1
  #修改gpinitsystem_config放开其中被注释掉的mirror相关设置,并修改其中 MIRROR_DATA_DIRECTORY=(/home/gpadmin/mirror)为 MIRROR_DATA_DIRECTORY=(/home/gpadmin/mirror /home/gpadmin/mirror) 具体设置几个由上面一条命令中的-n 参数来决定(如果是1就不用改这个） gpinitsystem -c gpinitsystem_config -s mdw1 -a source env.sh echo 'source /usr/local/greenplum-db/greenplum_path.sh' >> ~/.bashrc echo 'source env.sh' >> ~/.bashrc #不建议 开启gpadmin用户的远程访问（开启后是无密码的，尽快设置密码） #bash artifact/postinstall.sh #查看安装结果 ps -ef|grep postgres #查看集群状态，应该能看到两个master，4个Primary Segment 以及4个Mirror Segment gpstate -a #用下面命令查询各个节点的参数是否相同 gpssh -f hostfile -e 'ipcs -ls'
  
```
  
- 优化调整

```bash
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
```

- 初始化数据库以及创建用户

```
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

- nginx访问入口

```
  #nginx.cfg
  user  nginx;
  worker_processes  32;
  
  error_log  /var/log/nginx/error.log warn;
  pid        /var/run/nginx.pid;
  events {
      worker_connections  8192;
  #       muti_accept on;
          use epoll;
  }
  worker_rlimit_nofile 40000;
  #fastcgi_connect_timeout 100;
  #fastcgi_send_timeout 100;
  #fastcgi_read_timeout 100;
  #depends on nginx 1.11.2 and later
  stream {
  upstream gps {
      server mdw:5432 max_fails=3;
      server mdw1:5432 max_fails=3 backup;
      }
  
          server {
                  listen 5432;
                  proxy_pass gps;
          }
  }
  #createnginx.sh
  docker rm nginx -f
  docker run -itd -v /data/nginx/nginx.cfg:/etc/nginx/nginx.conf --name nginx -p 15432:5432 --restart=always cloud.lhcz.com:5000/nginx
```

## 安装配置服务端连接池

```
cd /tmp
#下载wget ftp://read:read@spwftp/linux/greenplum/pgbouncer_1.15_x86_64.tgz
tar zxvf pgbouncer_1.15_x86_64.tgz
./install.sh
mkdir bin
mv pgbouncer bin/
cd bin
```

- 创建配置文件

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

- transaction模式会造成datax报错，需要在datax的数据库连接字串中增加`prepareThreshold=0`
- 创建用户配置文件userlist.txt

```
"用户名" "密码明文"
"用户名" "md5+md5值"
```

- 编写启动脚本（容器内）

```
#path:/home/gpadmin/master/startpool.sh
cd /home/gpadmin/master/bin
./pgbouncer -d pgbouncer.ini
#进入控制台
psql -p 6543 -U hjdev pgbouncer
#输入密码
show help;显示帮助
```

- 编写“自启动”

```
#通过在/home/gpadmin/env.sh中追加语句实现变相自启动
bash /home/gpadmin/master/startpool.sh
```

- 使用

```
#进入连接池管理库
psql -p 6543 -h 127.0.0.1 -U hjdev pgbouncer
#输入密码
show help;#查看帮助
show clients;#查看连接的客户端
```