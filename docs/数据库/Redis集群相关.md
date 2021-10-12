# Redis集群相关

##  Redis常见问题

### 缓存雪崩

> 对于系统 A，假设每天高峰期每秒 5000 个请求，本来缓存在高峰期可以扛住每秒 4000 个请求，但是缓存机器意外发生了全盘宕机。缓存挂了，此时 1 秒 5000 个请求全部落数据库，数据库必然扛不住，它会报一下警，然后就挂了。此时，如果没有采用什么特别的方案来处理这个故障，DBA 很着急，重启数据库，但是数据库立马又被新的流量给打死了。这就是缓存雪崩。

#### 解决办法

+ 事前：Redis 高可用，主从+哨兵，Redis cluster，避免全盘崩溃。
+ 事中：本地 ehcache 缓存 + hystrix 限流&降级，避免 MySQL 被打死。
+ 事后：Redis 持久化，一旦重启，自动从磁盘上加载数据，快速恢复缓存数据。

### 缓存穿透

>  对于系统 A，假设一秒 5000 个请求，结果其中 4000 个请求是黑客发出的恶意攻击。黑客发出的那 4000 个攻击，缓存中查不到，每次你去数据库里查，也查不到。举个栗子。数据库 id 是从 1 开始的，结果黑客发过来的请求 id 全部都是负数。这样的话，缓存中不会有，请求每次都“**视缓存于无物**”，直接查询数据库。这种恶意攻击场景的缓存穿透就会直接把数据库给打死。

#### 解决办法

- 请求数据的 key 不存在于布隆过滤器中，可以确定数据就一定不会存在于数据库中，系统可以立即返回不存在。
- 请求数据的 key 存在于布隆过滤器中，则继续再向缓存中查询。

### 缓存击穿

> 缓存击穿，就是说某个 key 非常热点，访问非常频繁，处于集中式高并发访问的情况，当这个 key 在失效的瞬间，大量的请求就击穿了缓存，直接请求数据库，就像是在一道屏障上凿开了一个洞。

#### 解决办法

- 若缓存的数据是基本不会发生更新的，则可尝试将该热点数据设置为永不过期。
- 若缓存的数据更新不频繁，且缓存刷新的整个流程耗时较少的情况下，则可以采用基于 Redis、zookeeper 等分布式中间件的分布式互斥锁，或者本地互斥锁以保证仅少量的请求能请求数据库并重新构建缓存，其余线程则在锁释放后能访问到新缓存。
- 若缓存的数据更新频繁或者在缓存刷新的流程耗时较长的情况下，可以利用定时线程在缓存过期前主动地重新构建缓存或者延后缓存的过期时间，以保证所有的请求能一直访问到对应的缓存。



## 注意事项

+ 生产环境中禁止使用save命令手工保存数据，如果需要手工保存数据请使用bgsave。
+ 生产环境中必须设置访问密码。

## 单节点性能说明

单机的 Redis，能够承载的 QPS 大概就在上万到几万不等。对于缓存来说，一般都是用来支撑读高并发的。因此架构做成主从(master-slave)架构，一主多从，主负责写，并且将数据复制到其它的 slave 节点，从节点负责读。所有的读请求全部走从节点。这样也可以很轻松实现水平扩容，支撑读高并发。
![redis-master-slave](https://pic-1256216313.cos.ap-chengdu.myqcloud.com/img/redis-master-slave.png) 

Redis replication -> 主从架构 -> 读写分离 -> 水平扩容支撑读高并发。



## Redis主从集群主要机制

- Redis 采用**异步方式**复制数据到 slave 节点，不过 Redis2.8 开始，slave node 会周期性地确认自己每次复制的数据量；
- 一个 master node 是可以配置多个 slave node 的；
- slave node 也可以连接其他的 slave node；
- slave node 做复制的时候，不会 block master node 的正常工作；
- slave node 在做复制的时候，也不会 block 对自己的查询操作，它会用旧的数据集来提供服务；但是复制完成的时候，需要删除旧数据集，加载新数据集，这个时候就会暂停对外服务了；
- slave node 主要用来进行横向扩容，做读写分离，扩容的 slave node 可以提高读的吞吐量。
- **注意，如果采用了主从架构，那么建议必须开启 master node 的[持久化](https://github.com/doocs/advanced-java/blob/main/docs/high-concurrency/redis-persistence.md)，不建议用 slave node 作为 master node 的数据热备，因为那样的话，如果你关掉 master 的持久化，可能在 master 宕机重启的时候数据是空的，然后可能一经过复制， slave node 的数据也丢了。**
- 另外，master 的各种备份方案，也需要做。万一本地的所有文件丢失了，从备份中挑选一份 rdb 去恢复 master，这样才能**确保启动的时候，是有数据的**，即使采用了后续讲解的高可用机制，slave node 可以自动接管 master node，但也可能 sentinel 还没检测到 master failure，master node 就自动重启了，还是可能导致上面所有的 slave node 数据被清空。

## 哨兵集群主要机制（基本废弃，目前直接使用redis原生集群）

- 哨兵负责：集群监控、消息通知、故障转移、配置中心功能
- 哨兵至少需要 3 个实例，来保证自己的健壮性。
- 哨兵 + Redis 主从的部署架构，是**不保证数据零丢失**的，只能保证 Redis 集群的高可用性。
  - 主备切换的过程，可能会导致数据丢失
    - 因为 master->slave 的复制是异步的，所以可能有部分数据还没复制到 slave，master 就宕机了，此时这部分数据就丢失了。
    - 脑裂导致的数据丢失，某个 master 所在机器突然脱离了正常的网络，跟其他 slave 机器不能连接，但是实际上 master 还运行着。此时哨兵可能就会认为 master 宕机了，然后开启选举，将其他 slave 切换成了 master。这个时候，集群里就会有两个 master ，也就是所谓的脑裂。此时虽然某个 slave 被切换成了 master，但是可能 client 还没来得及切换到新的 master，还继续向旧 master 写数据。因此旧 master 再次恢复的时候，会被作为一个 slave 挂到新的 master 上去，自己的数据会清空，重新从新的 master 复制数据。而新的 master 并没有后来 client 写入的数据，因此，这部分数据也就丢失了。

- 对于哨兵 + Redis 主从这种复杂的部署架构，尽量在测试环境和生产环境，都进行充足的测试和演练。

### sdown 和 odown 转换机制

- sdown 是主观宕机，就一个哨兵如果自己觉得一个 master 宕机了，那么就是主观宕机
- odown 是客观宕机，如果 quorum 数量的哨兵都觉得一个 master 宕机了，那么就是客观宕机

sdown 达成的条件很简单，如果一个哨兵 ping 一个 master，超过了 `is-master-down-after-milliseconds` 指定的毫秒数之后，就主观认为 master 宕机了；如果一个哨兵在指定时间内，收到了 quorum 数量的其它哨兵也认为那个 master 是 sdown 的，那么就认为是 odown 了。

### 哨兵集群的自动发现机制

哨兵互相之间的发现，是通过 Redis 的 `pub/sub` 系统实现的，每个哨兵都会往 `__sentinel__:hello` 这个 channel 里发送一个消息，这时候所有其他哨兵都可以消费到这个消息，并感知到其他的哨兵的存在。

每隔两秒钟，每个哨兵都会往自己监控的某个 master+slaves 对应的 `__sentinel__:hello` channel 里**发送一个消息**，内容是自己的 host、ip 和 runid 还有对这个 master 的监控配置。

每个哨兵也会去**监听**自己监控的每个 master+slaves 对应的 `__sentinel__:hello` channel，然后去感知到同样在监听这个 master+slaves 的其他哨兵的存在。

每个哨兵还会跟其他哨兵交换对 `master` 的监控配置，互相进行监控配置的同步。

### quorum 和 majority

每次一个哨兵要做主备切换，首先需要 quorum 数量的哨兵认为 odown，然后选举出一个哨兵来做切换，这个哨兵还需要得到 majority 哨兵的授权，才能正式执行切换。

如果 quorum < majority，比如 5 个哨兵，majority 就是 3，quorum 设置为 2，那么就 3 个哨兵授权就可以执行切换。

但是如果 quorum >= majority，那么必须 quorum 数量的哨兵都授权，比如 5 个哨兵，quorum 是 5，那么必须 5 个哨兵都同意授权，才能执行切换。



## Redis 持久化的两种方式

+ RDB 持久化机制，是对 Redis 中的数据执行周期性的持久化。
  + RDB 会生成多个数据文件，每个数据文件都代表了某一个时刻中 Redis 的数据，这种多个数据文件的方式，非常适合做冷备，可以将这种完整的数据文件发送到一些远程的安全存储上去，比如说 Amazon 的 S3 云服务上去，在国内可以是阿里云的 ODPS 分布式存储上，以预定好的备份策略来定期备份 Redis 中的数据。
  + RDB 对 Redis 对外提供的读写服务，影响非常小，可以让 Redis 保持高性能，因为 Redis 主进程只需要 fork 一个子进程，让子进程执行磁盘 IO 操作来进行 RDB 持久化即可。
  + 相对于 AOF 持久化机制来说，直接基于 RDB 数据文件来重启和恢复 Redis 进程，更加快速。
  + 如果想要在 Redis 故障时，尽可能少的丢失数据，那么 RDB 没有 AOF 好。一般来说，RDB 数据快照文件，都是每隔 5 分钟，或者更长时间生成一次，这个时候就得接受一旦 Redis 进程宕机，那么会丢失最近 5 分钟（甚至更长时间）的数据。
  + RDB 每次在 fork 子进程来执行 RDB 快照数据文件生成的时候，如果数据文件特别大，可能会导致对客户端提供的服务暂停数毫秒，或者甚至数秒。
+ AOF 机制对每条写入命令作为日志，以 append-only 的模式写入一个日志文件中，在 Redis 重启的时候，可以通过回放 AOF 日志中的写入指令来重新构建整个数据集。
  + AOF 可以更好的保护数据不丢失，一般 AOF 会每隔 1 秒，通过一个后台线程执行一次 fsync 操作，最多丢失 1 秒钟的数据。
  + AOF 日志文件以 append-only 模式写入，所以没有任何磁盘寻址的开销，写入性能非常高，而且文件不容易破损，即使文件尾部破损，也很容易修复。
  + AOF 日志文件即使过大的时候，出现后台重写操作，也不会影响客户端的读写。因为在 rewrite log 的时候，会对其中的指令进行压缩，创建出一份需要恢复数据的最小日志出来。在创建新日志文件的时候，老的日志文件还是照常写入。当新的 merge 后的日志文件 ready 的时候，再交换新老日志文件即可。
  + AOF 日志文件的命令通过可读较强的方式进行记录，这个特性非常适合做灾难性的误删除的紧急恢复。比如某人不小心用 flushall 命令清空了所有数据，只要这个时候后台 rewrite 还没有发生，那么就可以立即拷贝 AOF 文件，将最后一条 flushall 命令给删了，然后再将该 AOF 文件放回去，就可以通过恢复机制，自动恢复所有数据。
  + 对于同一份数据来说，AOF 日志文件通常比 RDB 数据快照文件更大。
  + AOF 开启后，支持的写 QPS 会比 RDB 支持的写 QPS 低，因为 AOF 一般会配置成每秒 fsync 一次日志文件，当然，每秒一次 fsync ，性能也还是很高的。（如果实时写入，那么 QPS 会大降，Redis 性能会大大降低）。
  + 以前 AOF 发生过 bug，就是通过 AOF 记录的日志，进行数据恢复的时候，没有恢复一模一样的数据出来。所以说，类似 AOF 这种较为复杂的基于命令日志 merge 回放的方式，比基于 RDB 每次持久化一份完整的数据快照文件的方式，更加脆弱一些，容易有 bug。不过 AOF 就是为了避免 rewrite 过程导致的 bug，因此每次 rewrite 并不是基于旧的指令日志进行 merge 的，而是基于当时内存中的数据进行指令的重新构建，这样健壮性会好很多。
+ RDB 和 AOF 到底该如何选择
  + 不要仅仅使用 RDB，因为那样会导致你丢失很多数据；
  + 也不要仅仅使用 AOF，因为那样有两个问题：第一，你通过 AOF 做冷备，没有 RDB 做冷备来的恢复速度更快；第二，RDB 每次简单粗暴生成数据快照，更加健壮，可以避免 AOF 这种复杂的备份和恢复机制的 bug；
  + Redis 支持同时开启开启两种持久化方式，我们可以综合使用 AOF 和 RDB 两种持久化机制，用 AOF 来保证数据不丢失，作为数据恢复的第一选择；用 RDB 来做不同程度的冷备，在 AOF 文件都丢失或损坏不可用的时候，还可以使用 RDB 来进行快速的数据恢复。



## 各种搭建

### 单节点

配置文件single.conf

```properties
#RDB持久化配置
#在300秒(5分钟)之后，如果至少有10个key发生变化，则dump内存快照。
save 300 10
#压缩
rdbcompression yes
#AOF持久化配置
## 只有在“yes”下，aof重写/文件同步等特性才会生效  
appendonly yes
#每秒钟同步一次，该策略为AOF的缺省策略。
appendfsync everysec
## 指定aof文件名称  
appendfilename appendonly.aof  
#数据文件夹
dir /data
#设置访问密码
requirepass yourpassword
#允许外网访问（监听0.0.0.0），（在没有设置密码且没有设置bind ip时候默认开启只监听127.0.0.1）
protected-mode no
```

容器方式启动

```bash
mkdir data
docker run -it --restart=always -p 6379:6379 --name redis -v `pwd`/data:/data -v `pwd`/single.conf:/etc/redis.conf redis redis-server /etc/redis.conf
```

### 集群方式

+ Redis的集群不支持nat类型的网络，所以使用容器搭建的话需要用主机网络的方式接入`--net host`
+ Redis镜像版本>5建议直接使用最新版本镜像
+ 集群依赖两个tcp端口默认为x=6379 y=x+10000=16379 如果使用防火墙需要注意开启这两个端口的出入栈规则。

配置文件cluster.conf 

```properties
#启用集群模式
cluster-enabled yes
#集群自动配置文件（非用户自定义而是集群自己使用的）
cluster-config-file cluster.conf
#集群节点最大超时时间单位为毫秒，如某个节点10秒没响应将视为掉线
cluster-node-timeout 10000
#副本节点有效性校验因子（结合节点超时时间使用，当节点与主节点上次同步时间>(节点超时时间*校验因子)后认为此备份无效，不参与主节点故障转移
cluster-slave-validity-factor 10
#主节点将保持连接的最小副本数
cluster-migration-barrier 3
#如果将其设置为 yes，默认情况下，如果任何节点未覆盖一定百分比的密钥空间，则集群将停止接受写入。
#如果该选项设置为 no，即使只能处理有关密钥子集的请求，集群仍将提供查询服务。
cluster-require-full-coverage no
#如果设置为 no，默认情况下，当集群被标记为失败时，Redis 集群中的节点将停止服务所有流量，或者当节点无法访问时达到法定人数或未达到完全覆盖时。这可以防止从不知道集群中的变化的节点读取可能不一致的数据。可以将此选项设置为 yes 以允许在失败状态期间从节点读取，这对于希望优先考虑读取可用性但仍希望防止不一致写入的应用程序非常有用。它也可以用于使用只有一个或两个分片的 Redis Cluster 时，因为它允许节点在主节点发生故障但无法自动故障转移时继续提供写入服务。
cluster-allow-reads-when-down yes
requirepass yourpassword
#从节点连接主节点的密码
masterauth yourpassword
#保存的间隔时间和至少变动key数
save 600 10
```

```bash
#挨个启动每个节点（需要提前将配置文件以及存储路径创建好），我们建立6个节点，其中3个做为主节点（存数据），三个作为从节点，数据在3个主节点之间均分（便于）
docker run -itd --name redis1 --restart=always --net host -v `pwd`/data:/data -v `pwd`/cluster.conf:/etc/redis.conf -p 6379:6379 redis redis-server /etc/redis.conf
#创建其他节点……
#进入节点准备建立集群连接
docker exec -it redis1 bash
#集群连接不支持主机名，只能使用ip，所以每个节点的ip地址需要固定
redis-cli -a yourpassword --cluster create 172.21.0.2:6379 172.21.0.3:6379 172.21.0.4:6379 172.21.0.5:6379 172.21.0.6:6379 172.21.0.7:6379 --cluster-replicas 1
#输入yes进行确认
#输出
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```



```bash
#其他常用集群命令
#fix命令来修复集群，以便根据每个节点是否具有权威性的哈希槽来迁移密钥。
redis-cli --cluster fix
#check检查集群状态
redis-cli --cluster check
#添加主节点add-node后第一个参数为新节点的ip和端口，第二个参数为现有集群中的任意一个ip和端口
redis-cli -a yourpassword --cluster add-node 172.21.0.8:6379 172.21.0.2:6379
#添加从节点（可以通过增加--cluster-master-id 对应的主节点id）来指定其成为特定节点的副本，具体节点id可以在节点配置文件（自动）中获取到
redis-cli -a yourpassword --cluster add-node 172.21.0.9:6379 172.21.0.2:6379 --cluster-slave
#删除节点del-node后第一个参数为集群中任意一个节点ip和端口，第二个参数为要删除的节点id
redis-cli --cluster del-node 127.0.0.1:7000 `<node-id>`
```



## 常用配置

```yaml
#集群使用 开始
#无磁盘化复制master 在内存中直接创建 RDB ，然后发送给 slave，不会在自己本地落地磁盘
repl-diskless-sync yes
# 等待 5s 后再开始复制，因为要等更多 slave 重新连接过来
repl-diskless-sync-delay 5
#要求至少有 1 个 slave完成写入
min-slaves-to-write 1
#数据复制和同步的延迟不能超过 10 秒，如果说一旦所有的 slave，数据复制和同步的延迟都超过了 10 秒钟，那么这个时候，master 就不会再接收任何请求了。
min-slaves-max-lag 10
#集群使用 结束
#日志配置
loglevel warning
```

