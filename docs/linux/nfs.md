# 针对大数据组件的NFS挂载参数

+ nfs挂载的远程磁盘如果用于kafka、zookeeper、hadoop 之类的组件需要增加 `nolock`属性