# mysql删除大表
> mysql某个表中数据量太大会导致ddl语句执行不动的问题（例如单表过亿），通过linux的文件硬链接引用数对删除的特性来快速删除表；（具体原理为当一个文件的硬引用数没有归零时候，系统不会真正的去删除文件，相当于逻辑删除，这样对mysql数据库管理系统来说删除文件将非常快，等数据库层面文件操作完成了，我们再从系统层面去处理这个硬链接的文件即可。）

```bash
#sql
create table t1_new like t1;
rename table t1 to t1_deleted,t1_new to t1;
#bash命令
#进到mysql数据文件夹
cd /data/mysql_data/sbtest/
#建立硬链接
ln t1_deleted.ibd t1_deleted.ibd.hdlk
#sql
drop table t1_deleted;
#bash命令 快速清空文件
truncate -s 1 t1_deleted.ibd.hdlk
rm -f t1_deleted.ibd.hdlk
```

至此任务完成