# flink集群搭建

## 集群类型
+ 单机伪集群 ：一般为开始测试使用
+ standalone独立集群 ：不依赖hadoop环境即可使用，需要在每台机器安装配置flink环境
+ yarn集群 ： 一般生产环境使用，可以使用yarn资源，只需要在一台机器运行flink即可