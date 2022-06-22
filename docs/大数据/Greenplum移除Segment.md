# Greenplum移除Segment
刚才登陆上来看到以前写的一篇文章扩展greenplum集群 下面有人回帖问如何裁剪SEGMENT。因为并没有看到GREENPLUM提供什么工具可以去除某些已经在使用的SEGMENT的。刚刚去试了一下，好像可以，只能说好像可以。下面把操作步骤贴出来，大家也可以一起看看有没有什么问题。

下面是现有集群的一个大致情况

testutf8=# select * from gp_configuration;
content | definedprimary | dbid | isprimary | valid | hostname | port  |             datadir
———+—————-+——+———–+——-+———-+——-+———————————-
-1 | t              |    1 | t         | t     | hadoop7  |  5345 | /home/gpadmin1/gp3master/aligp-1
0 | f              |    8 | f         | t     | hadoop8  | 30001 | /home/gpadmin1/gp3datam1/aligp0
1 | f              |    9 | f         | t     | hadoop8  | 30002 | /home/gpadmin1/gp3datam2/aligp1
0 | t              |    2 | t         | t     | hadoop7  | 20001 | /home/gpadmin1/gp3datap1/aligp0
1 | t              |    3 | t         | t     | hadoop7  | 20002 | /home/gpadmin1/gp3datap2/aligp1
3 | t              |    5 | t         | t     | hadoop8  | 20002 | /home/gpadmin1/gp3datap2/aligp3
-1 | f              |   14 | f         | t     | hadoop8  |  5345 | /home/gpadmin1/gp3master/aligp-1
2 | t              |    4 | t         | t     | hadoop8  | 20001 | /home/gpadmin1/gp3datap1/aligp2
2 | f              |   10 | f         | t     | hadoop9  | 30001 | /home/gpadmin1/gp3datam1/aligp2
3 | f              |   11 | f         | t     | hadoop9  | 30002 | /home/gpadmin1/gp3datam2/aligp3
4 | f              |   12 | f         | t     | hadoop7  | 30001 | /home/gpadmin1/gp3datam1/aligp4
5 | f              |   13 | f         | t     | hadoop7  | 30002 | /home/gpadmin1/gp3datam2/aligp5
4 | t              |    6 | t         | t     | hadoop9  | 20001 | /home/gpadmin1/gp3datap1/aligp4
5 | t              |    7 | t         | t     | hadoop9  | 20002 | /home/gpadmin1/gp3datap2/aligp5 –标为红色的部分即为要删除的SEGMENT
(14 rows)
–创建两张表t1，t2.在后面的操作中我不会对t1进行特殊处理，因此理论上删除某个SEGMENT以后它上面的数据会有所减少。而对于t2，我会在修改集群配置以前把在要被删除的那些SEGMENT上的数据手动迁移到其他保留的SEGMENT上。
testutf8=# create table t1 as select nextval(’seq1′) from pg_class;
NOTICE:  Table doesn’t have ‘DISTRIBUTED BY’ clause — Using column(s) named ‘nextval’ as the Greenplum Database data distribution key for this table.
HINT:  The ‘DISTRIBUTED BY’ clause determines the distribution of data. Make sure column(s) chosen are the optimal data distribution key to minimize skew.
SELECT 267
testutf8=# select gp_segment_id,count(1) from t1 group by 1;
gp_segment_id | count
—————+——-
1 |    43
4 |    48
2 |    42
0 |    44
3 |    41
5 |    49
(6 rows)

testutf8=# create table t2 as select nextval(’seq1′) from pg_class;
NOTICE:  Table doesn’t have ‘DISTRIBUTED BY’ clause — Using column(s) named ‘nextval’ as the Greenplum Database data distribution key for this table.
HINT:  The ‘DISTRIBUTED BY’ clause determines the distribution of data. Make sure column(s) chosen are the optimal data distribution key to minimize skew.
SELECT 268
testutf8=# select gp_segment_id,count(1) from t2 group by 1;
gp_segment_id | count
—————+——-
1 |    44
2 |    43
0 |    41
4 |    49
3 |    42
5 |    49
(6 rows)

下面几步所做的就是对t2表的在将被删除的SEGMENT上的数据进行迁移

#connect to hadoop9:20001 from hadoop9
[gpadmin1@hadoop9 ~]$ PGOPTIONS=”-c gp_session_role=utility” psql -h hadoop9 -p 20001 -U r1

psql (8.2.13)
Type “help” for help.

testutf8=#
testutf8=# select count(*) from t2;
count
——-
49
(1 row)

testutf8=# select * from t2 limit 1;        –这里随便查出一条数据，在集群相应的SEGMENT被去除以后，可以用来验证这条数据是否还存在集群中
nextval
———
810
(1 row)

testutf8=# copy t2 to ‘/home/gpadmin1/hadoop9_20001_t2′;     –通过COPY命令把t2在此SEGMENT上的数据备份出来
COPY 49

#connect to hadoop9:20002 from hadoop7
[gpadmin1@hadoop7 ~]$ PGOPTIONS=”-c gp_session_role=utility” psql -h hadoop9 -p 20002 -U r1
psql (8.2.13)
Type “help” for help.

testutf8=# select count(*) from t2;
count
——-
49
(1 row)

testutf8=# select * from t2 limit 1;
nextval
———
811
(1 row)

testutf8=# copy t2 to ‘/home/gpadmin1/hadoop9_20002_t2′;
COPY 49

[gpadmin1@hadoop9 ~]$ ls
1.sh        conf         gp3datam1  gp3datap1  gp3master  gp4datam2  gp4datap2    hadoop9_20001_t2 hy         pgback    t3

cleangp.sh  cpgp_cfg.sh  gp3datam2  gp3datap2  gp4datam1  gp4datap1  gpAdminLogs  hadoop9_20002_t2 joe.wangh  software  tools
[gpadmin1@hadoop9 ~]$ PGOPTIONS=”-c gp_session_role=utility” psql -h hadoop7 -p 20001 -U r1
psql (8.2.13)
Type “help” for help.
testutf8=# copy t2 from ‘/home/gpadmin1/hadoop9_20001_t2′;
ERROR:  could not open file “/home/gpadmin1/hadoop9_20001_t2″ for reading: No such file or directory
testutf8=# /q

–这里因为我连接到的是HADOOP7:20001的SEGMENT,而刚才两个COPY出来的文件都在HADOOP9上，因此需要SCP到HADOOP7上进行应用。
[gpadmin1@hadoop9 ~]$ scp hadoop9_20001_t2 hadoop7:/home/gpadmin1/
hadoop9_20001_t2                                                                                  100%  211     0.2KB/s   00:00
[gpadmin1@hadoop9 ~]$ scp hadoop9_20002_t2 hadoop7:/home/gpadmin1/
hadoop9_20002_t2                                                                                  100%  212     0.2KB/s   00:00
[gpadmin1@hadoop9 ~]$ PGOPTIONS=”-c gp_session_role=utility” psql -h hadoop7 -p 20001 -U r1
psql (8.2.13)
Type “help” for help.

testutf8=# copy t2 from ‘/home/gpadmin1/hadoop9_20001_t2′;
COPY 49
testutf8=# copy t2 from ‘/home/gpadmin1/hadoop9_20002_t2′;
COPY 49
testutf8=# select count(*) from t2;
count
——-
139               –从前面t2表在各个segment上分布的情况可以看到，41+49+49=139，正好是gp_segment_id为0，4，5之和
(1 row)

OK，下面我们可以把数据库正常关闭了。

[gpadmin1@hadoop7 pg_log]$ gpstop
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-Starting gpstop with args: ”
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-Gathering information and validating the environment…
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-Obtaining Greenplum Master catalog information
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-Obtaining Segment details from master…
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-Greenplum Version: ‘postgres (Greenplum Database) 3.3.7.0 build 1′
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:———————————————
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-Master instance parameters
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:———————————————
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-Master Greenplum instance process active PID = 22683
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-Database                                     = template1
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-Master Port                                  = 5345
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-Master directory                             = /home/gpadmin1/gp3master/aligp-1
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-Shutdown mode                                = smart
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-Shutdown Master standby host                 = On
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:———————————————
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-Segment instances that will be shutdown:
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:———————————————
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-Host     Datadir                          Port   Status
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-hadoop7  /home/gpadmin1/gp3datap1/aligp0  20001  Valid
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-hadoop8  /home/gpadmin1/gp3datam1/aligp0  30001  Valid
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-hadoop7  /home/gpadmin1/gp3datap2/aligp1  20002  Valid
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-hadoop8  /home/gpadmin1/gp3datam2/aligp1  30002  Valid
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-hadoop8  /home/gpadmin1/gp3datap1/aligp2  20001  Valid
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-hadoop9  /home/gpadmin1/gp3datam1/aligp2  30001  Valid
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-hadoop8  /home/gpadmin1/gp3datap2/aligp3  20002  Valid
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-hadoop9  /home/gpadmin1/gp3datam2/aligp3  30002  Valid
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-hadoop9  /home/gpadmin1/gp3datap1/aligp4  20001  Valid
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-hadoop7  /home/gpadmin1/gp3datam1/aligp4  30001  Valid
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-hadoop9  /home/gpadmin1/gp3datap2/aligp5  20002  Valid
20101127:20:52:19:gpstop:hadoop7:gpadmin1-[INFO]:-hadoop7  /home/gpadmin1/gp3datam2/aligp5  30002  Valid

Continue with Greenplum instance shutdown Yy|Nn (default=N):
> y
20101127:20:52:20:gpstop:hadoop7:gpadmin1-[INFO]:-There are 0 connections to the database
20101127:20:52:20:gpstop:hadoop7:gpadmin1-[INFO]:-Commencing Master instance shutdown with mode=’smart’
20101127:20:52:20:gpstop:hadoop7:gpadmin1-[INFO]:-Master host=hadoop7
20101127:20:52:20:gpstop:hadoop7:gpadmin1-[INFO]:-Commencing Master instance shutdown with mode=smart
20101127:20:52:20:gpstop:hadoop7:gpadmin1-[INFO]:-Master segment instance directory=/home/gpadmin1/gp3master/aligp-1
20101127:20:52:21:gpstop:hadoop7:gpadmin1-[INFO]:-Stopping gpsyncmaster on standby host hadoop8 mode=fast
20101127:20:52:22:gpstop:hadoop7:gpadmin1-[INFO]:-Successfully shutdown sync process on hadoop8
20101127:20:52:22:gpstop:hadoop7:gpadmin1-[INFO]:-Commencing parallel segment instance shutdown, please wait…
…..
20101127:20:52:27:gpstop:hadoop7:gpadmin1-[INFO]:————————————————-
20101127:20:52:27:gpstop:hadoop7:gpadmin1-[INFO]:-Parallel process exit status
20101127:20:52:27:gpstop:hadoop7:gpadmin1-[INFO]:————————————————-
20101127:20:52:27:gpstop:hadoop7:gpadmin1-[INFO]:-Total processes marked as completed         = 12
20101127:20:52:27:gpstop:hadoop7:gpadmin1-[INFO]:-Total processes marked as failed            = 0
20101127:20:52:27:gpstop:hadoop7:gpadmin1-[INFO]:————————————————-
20101127:20:52:27:gpstop:hadoop7:gpadmin1-[INFO]:-Total instances marked invalid and bypassed = 0
20101127:20:52:27:gpstop:hadoop7:gpadmin1-[INFO]:-Successfully shutdown 12 of 12 segment instances
20101127:20:52:27:gpstop:hadoop7:gpadmin1-[INFO]:-Database successfully shutdown with no errors reported

然后使用-m参数打开数据库，更改数据字典表信息。主要更改gp_configuration和gp_id两个数据字典。一开始我只更改了gp_configuration里的信息，后来在启动的时候报错，再把gp_id里面信息更改一下就可以了。
[gpadmin1@hadoop7 pg_log]$ gpstart -m
20101127:20:53:04:gpstart:hadoop7:gpadmin1-[INFO]:-Starting gpstart with args: ‘-m’
20101127:20:53:04:gpstart:hadoop7:gpadmin1-[INFO]:-Gathering information and validating the environment…
20101127:20:53:04:gpstart:hadoop7:gpadmin1-[INFO]:-local Greenplum Version: ‘postgres (Greenplum Database) 3.3.7.0 build 1′
20101127:20:53:04:gpstart:hadoop7:gpadmin1-[INFO]:-Starting Master instance in admin mode
20101127:20:53:05:gpstart:hadoop7:gpadmin1-[INFO]:-Obtaining Greenplum Master catalog information
20101127:20:53:05:gpstart:hadoop7:gpadmin1-[INFO]:-Obtaining Segment details from master…
20101127:20:53:05:gpstart:hadoop7:gpadmin1-[INFO]:-Master Started…
[gpadmin1@hadoop7 pg_log]$ PGOPTIONS=”-c gp_session_role=utility” psql
psql (8.2.13)
Type “help” for help.

testutf8=# delete from gp_configuration where content=5;
DELETE 2
testutf8=# delete from gp_configuration where content=4;
DELETE 2
testutf8=# select * from gp_id;
gpname | numsegments | dbid | content
——–+————-+——+———
GP     |           6 |    1 |      -1
(1 row)

testutf8=# update gp_id set numsegments=4;
UPDATE 1
testutf8=# /q
[gpadmin1@hadoop7 pg_log]$ gpstop -m
20101127:20:53:46:gpstop:hadoop7:gpadmin1-[INFO]:-Starting gpstop with args: ‘-m’
20101127:20:53:46:gpstop:hadoop7:gpadmin1-[INFO]:-Gathering information and validating the environment…
20101127:20:53:46:gpstop:hadoop7:gpadmin1-[INFO]:-Obtaining Greenplum Master catalog information
20101127:20:53:46:gpstop:hadoop7:gpadmin1-[INFO]:-Obtaining Segment details from master…
20101127:20:53:46:gpstop:hadoop7:gpadmin1-[INFO]:-Greenplum Version: ‘postgres (Greenplum Database) 3.3.7.0 build 1′
20101127:20:53:46:gpstop:hadoop7:gpadmin1-[INFO]:-There are 0 connections to the database
20101127:20:53:46:gpstop:hadoop7:gpadmin1-[INFO]:-Commencing Master instance shutdown with mode=’smart’
20101127:20:53:46:gpstop:hadoop7:gpadmin1-[INFO]:-Master host=hadoop7
20101127:20:53:46:gpstop:hadoop7:gpadmin1-[INFO]:-Commencing Master instance shutdown with mode=smart
20101127:20:53:46:gpstop:hadoop7:gpadmin1-[INFO]:-Master segment instance directory=/home/gpadmin1/gp3master/aligp-1

关键时刻，以正常模式启动数据库试试，胜负在此一举。
[gpadmin1@hadoop7 pg_log]$ gpstart
20101127:20:54:05:gpstart:hadoop7:gpadmin1-[INFO]:-Starting gpstart with args: ”
20101127:20:54:05:gpstart:hadoop7:gpadmin1-[INFO]:-Gathering information and validating the environment…
20101127:20:54:05:gpstart:hadoop7:gpadmin1-[INFO]:-local Greenplum Version: ‘postgres (Greenplum Database) 3.3.7.0 build 1′
20101127:20:54:05:gpstart:hadoop7:gpadmin1-[INFO]:-Starting Master instance in admin mode
20101127:20:54:06:gpstart:hadoop7:gpadmin1-[INFO]:-Obtaining Greenplum Master catalog information
20101127:20:54:06:gpstart:hadoop7:gpadmin1-[INFO]:-Obtaining Segment details from master…
20101127:20:54:06:gpstart:hadoop7:gpadmin1-[INFO]:-Master Started…
20101127:20:54:06:gpstart:hadoop7:gpadmin1-[INFO]:-Shutting down master
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:—————————
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:-Master instance parameters
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:—————————
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:-Database                 = template1
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:-Master Port              = 5345
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:-Master directory         = /home/gpadmin1/gp3master/aligp-1
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:-Master standby start     = On
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:—————————————
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:-Segment instances that will be started
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:—————————————
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:-Host     Datadir                          Port   Status
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:-hadoop7  /home/gpadmin1/gp3datap1/aligp0  20001  Valid
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:-hadoop8  /home/gpadmin1/gp3datam1/aligp0  30001  Valid
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:-hadoop7  /home/gpadmin1/gp3datap2/aligp1  20002  Valid
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:-hadoop8  /home/gpadmin1/gp3datam2/aligp1  30002  Valid
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:-hadoop8  /home/gpadmin1/gp3datap1/aligp2  20001  Valid
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:-hadoop9  /home/gpadmin1/gp3datam1/aligp2  30001  Valid
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:-hadoop8  /home/gpadmin1/gp3datap2/aligp3  20002  Valid
20101127:20:54:07:gpstart:hadoop7:gpadmin1-[INFO]:-hadoop9  /home/gpadmin1/gp3datam2/aligp3  30002  Valid

Continue with Greenplum instance startup Yy|Nn (default=N):
> y
20101127:20:54:09:gp.py:hadoop7:gpadmin1-[INFO]:-Starting standby master
20101127:20:54:09:gp.py:hadoop7:gpadmin1-[INFO]:-Checking if standby master is running on host: hadoop8  in directory: /home/gpadmin1/gp3master/aligp-1
20101127:20:54:09:gp.py:hadoop7:gpadmin1-[INFO]:-No db instance process, entering recovery startup mode
20101127:20:54:09:gpstart:hadoop7:gpadmin1-[INFO]:-Commencing parallel segment instance startup, please wait…
…..
20101127:20:54:14:gpstart:hadoop7:gpadmin1-[INFO]:-Process results…
20101127:20:54:14:gpstart:hadoop7:gpadmin1-[INFO]:—————————————————–
20101127:20:54:14:gpstart:hadoop7:gpadmin1-[INFO]:-Total processes marked as completed             = 8
20101127:20:54:14:gpstart:hadoop7:gpadmin1-[INFO]:-Total processes marked as failed                = 0
20101127:20:54:14:gpstart:hadoop7:gpadmin1-[INFO]:—————————————————–
20101127:20:54:14:gpstart:hadoop7:gpadmin1-[INFO]:-Total instances marked invalid and bypassed     = 0
20101127:20:54:14:gpstart:hadoop7:gpadmin1-[INFO]:-Successfully started 8 of 8 segment instances
20101127:20:54:14:gpstart:hadoop7:gpadmin1-[INFO]:—————————————————–
20101127:20:54:14:gpstart:hadoop7:gpadmin1-[INFO]:-Starting Master instance hadoop7 directory /home/gpadmin1/gp3master/aligp-1
20101127:20:54:15:gpstart:hadoop7:gpadmin1-[INFO]:-Command pg_ctl reports Master hadoop7 instance active
NOTICE:  Master mirroring synchronizing
20101127:20:54:16:gpstart:hadoop7:gpadmin1-[INFO]:-Database successfully started with no errors reported

正常启动了，现在连接上去看看刚才t1和t2表的情况如何。
[gpadmin1@hadoop7 pg_log]$ psql
psql (8.2.13)
Type “help” for help.
–这里可以看到，t1表在0，1，2，3这4个SEGMENT上的数据量没变化，也就是说在被删除的两个SEGMENT上的数据丢失了。
testutf8=# select gp_segment_id,count(1) from t1 group by 1;
gp_segment_id | count
—————+——-
0 |    44
1 |    43
2 |    42
3 |    41
(4 rows)
–在0号SEGMENT上数据量是139，也就是前面我们所说的41+49+49。
testutf8=# select gp_segment_id,count(1) from t2 group by 1;
gp_segment_id | count
—————+——-
3 |    42
0 |   139
1 |    44
2 |    43
(4 rows)

下面再查询一下刚才提取的t2表在被删除SEGMENT上的两个数值，看看它们是否依然可以正常被查询到。
testutf8=# /d t2
Table “public.t2″
Column  |  Type  | Modifiers
———+——–+———–
nextval | bigint |
Distributed by: (nextval)
–t2表上正常
testutf8=# select * from t2 where nextval=810;
nextval
———
810
(1 row)

testutf8=# select * from t2 where nextval=811;
nextval
———
811
(1 row)
–t1表上已经查不到了
testutf8=# select * from t1 where nextval=811;
nextval
———
(0 rows)

testutf8=# select * from t1 where nextval=810;
nextval
———
(0 rows)

当然，这种从GREENPLUM集群中去除SEGMENT的方法不一定是完全正确的，纯属个人YY一下然后试了试而已。

也许内部还是一些细节是我没有考虑到的，刚才我在这个去除了SEGMENT的集群中做了一些功能测试，结果还都是挺正常的，但是保不准哪一天他会出问题。

 

补充两点

1、在从gp_configuration里面删除相关信息以后，要保证剩余的SEGMENT的CONTENT列和DBID列是连续的。不然在接下 来gpstop -m的时候会报错index out of list

2、更改完MASTER上的gp_id以后，还要连接到各个SEGMENT上更改gp_id。如果你在前面的操作中更改了DBID和CONTENT 的话，那么在更改gp_id的时候也需要做相应的更改。

