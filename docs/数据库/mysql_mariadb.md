# mysql/mariadb

###  注意事项

- mariadb/mysql 生产库一定要设置innodb引擎的内存，默认值很小影响效率，还可以根据是否高写入或高读取调整写盘时间