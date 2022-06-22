* mariadb/mysql 生产库一定要设置innodb引擎的内存，默认值很小影响效率，还可以根据是否高写入或高读取调整写盘时间
* 如果要创建的表结构行长度大于8126会报错需要设置:
```
innodb_file_per_table
innodb_file_format = Barracuda
innodb_strict_mode=0
```