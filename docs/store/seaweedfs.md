# seaweedfs

> 简介:seaweedfs是一套基于go语言开发的分布式存储系统，目前最佳的海量小文件存储解决方案（大中型文件存储也没问题），支持fuse加载、支持api方式调用、支持设置有效期自动清理、支持多机房多机柜异地容灾等,项目地址：https://github.com/chrislusf/seaweedfs

## 搭建

```bash
#最小化集群（master+volume+filer）
./weed server -dir data/vol1 -filer -ip localhost -master.dir data/mdir -volume.images.fix.orientation -volume.max 5000 - volume.port 9334
#在另外的机器加一个vol节点
./weed volume -dir data/vol2 -images.fix.orientation -port 9335 -ip localhost -max 5000 
#在另外的机器加一个master节点（master节点需要单数个）
./weed master -ip localhost -mdir data/mdir -port 9333 -peers masterip1:9333,masterip2:9333
#在另外机器加filer节点
./weed filer -ip localhost -master masterip1:9333,masterip2:9333 -peers filerip1:8888,filerip2:8888 -port 8888
```

## 注意事项

- volume的max参数代表最大可创建的块数量，一个块默认为30GB，根据实例所在服务器的磁盘空间计算，保留必要的冗余之后可以尽可能多的设置。
- 使用seaweedfs的filer组件存储图片如果出现502、503、各种timeout的错误在排除网络带宽、filer.toml设置的最大数据库连接数 的因素后可以尝试分流filer服务的流量，因为根据seaweedfs的wiki中说的filer要处理文件内容并完成网络请求的分发所以压力比较大，所以适时地扩充更多的filer节点能有效的降低或避免50x以及timeout的发生

## 使用

```bash
#curl通过filer上传图片
curl http://ipaddr:8888/testjpg.jpg -F "img=@/tmp/testjpg.jpg"
#curl通过filer上传图片图片有效期1年
curl http://ipaddr:8888/testjpg.jpg?ttl=1y -F "img=@/tmp/testjpg.jpg"

#不使用filer上传
curl http://ipaddr:9333/dir/assign
#不使用filer上传图片有效期1个月
curl http://ipaddr:9333/dir/assign?ttl=1m
#返回:
{"fid":"123,12312321312","url":"ipaddrreturnbymaster:9334","publicUrl":"ipaddr:9334","count":1}
#上传文件
curl http://ipaddrreturnbymaster:9334/123,12312321312 -F "img=@/tmp/testjpg.jpg"
#通过filer将文件挂载到本地操作
weed mount -filer=localhost:8888 -dir=/some/existing/dir -filer.path=/one/remote/folder
```

- 通过kotlin调用（java类似）

```kotlin
//使用javaclient操作seaweedfs filer接口
//kotlin+hutool+seaweedfs-javaclient
@Test
fun filerTest(){
    //connect to filer grpc port
    val fc=FilerClient("localhost",18888)
    //准备数据
    val ba15mb= RandomUtils.nextBytes(1024*1024*15)
    val ba1mb=RandomUtils.nextBytes(1024*1024)
    val ba50mb=RandomUtils.nextBytes(1024*1024*50)
    //清理测试目录
    fc.rm("/ice",true,true)
    fc.mkdirs("/ice",666)
    for (i in 0 until 10){
        var sos1:SeaweedOutputStream
        var sos15:SeaweedOutputStream
        var sos50:SeaweedOutputStream
        var ut= measureTimeMillis {
            sos1=SeaweedOutputStream(fc,"/ice/idx_${i}_1mb.dat")
            sos15=SeaweedOutputStream(fc,"/ice/idx_${i}_15mb.txt")
            sos50=SeaweedOutputStream(fc,"/ice/idx_${i}_50mb.txt")
        }
        println("ut1:${ut} ms")
        ut= measureTimeMillis {
            sos1?.use {
                it.write(ba1mb)
            }
            sos15?.use {
                it.write(ba15mb)
            }
            sos50?.use {
                it.write(ba50mb)
            }
        }
        println("ut2:${ut} ms")
    }

    println("ok")
}
//测试结果：ut1:20 ms
//ut2:7104 ms
//ut1:1 ms
//ut2:683 ms
//ut1:0 ms
//ut2:691 ms
//ut1:0 ms
//ut2:632 ms
//ut1:0 ms
//ut2:738 ms
//ut1:0 ms
//ut2:720 ms
//ut1:0 ms
//ut2:655 ms
//ut1:1 ms
//ut2:658 ms
//ut1:1 ms
//ut2:667 ms
//ut1:0 ms
//ut2:634 ms
//ok
```

## 性能优化

- 增加并发写入支持

```bash
#并发写入12个volume复制份数为2实际写入为24个volume并发写入（同时写入12个不同的）
curl http://localhost:9333/vol/grow?count=12&replication=001
```

- 增加并发读取支持

```bash
#设置复制份数为2则同一个文件支持两个并发读取，不同的资源并发读取数量以volume分块数为准（非volume server节点数）
curl http://localhost:9333/vol/grow?count=12&replication=001
```

- 增加同时打开文件数量

```bash
#先执行这个命令
ulimit -n 10240
#再启动seaweedfs
./weed .....
```

## 问题处理

- 修复丢失volume（有复制份数的情况下）,生产系统建议周期性的定时执行此任务

```
weed shell -master masterip:9333
volume.fix.replication
```