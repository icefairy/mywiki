# Redis相关

## 惯例

+ 在String、Hashes、Lists、Sets、Sorted Sets几个常用数据类型中结合业务选择合适的类型来存储。

## 常用命令

```bash
#连接集群（增加-c参数）
redis-cli -a password -c -h 127.0.0.1

```



## 特性

### 过期策略

+ 定时删除
  + 默认Redis每100MS随机抽取一部分key对其进行检测是否过期需要删除。
  + 不全部执行的原因在于数据较多时候全部检测会耗费太多CPU，无法保证性能。
+ 惰性(LRU)删除
  + 在用户获取某个key的时候会检测是否过期，如果过期了删除并返回空。
+ 数据残留问题
  + 上述两种策略无法对全部数据进行过期删除，可能会造成实际的可用内存空间浪费，此时应配合内存淘汰机制配置。

### 内存淘汰机制

+ noeviction: 当内存不足以容纳新写入数据时，新写入操作会报错，这个一般没人用吧，实在是太恶心了。
+ allkeys-lru：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的 key（这个是最常用的）。
+ allkeys-random：当内存不足以容纳新写入数据时，在键空间中，随机移除某个 key，这个一般没人用吧，为啥要随机，肯定是把最近最少使用的 key 给干掉啊。
+ volatile-lru：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的 key（这个一般不太合适）。
+ volatile-random：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个 key。
+ volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的 key 优先移除。

## 数据类型及特点

### String

string 是 redis 最基本的类型，你可以理解成与 Memcached 一模一样的类型，一个 key 对应一个 value。

string 类型是二进制安全的。意思是 redis 的 string 可以包含任何数据。比如jpg图片或者序列化的对象。

string 类型是 Redis 最基本的数据类型，string 类型的值最大能存储 512MB。

常用命令：set、get、decr、incr、mget等。

注意：一个键最大能存储512MB。

### Hashes

Redis hash 是一个键值(key=>value)对集合；是一个 string 类型的 field 和 value 的映射表，hash 特别适合用于存储对象（对象不能嵌套其他对象）。

每个 hash 可以存储 232 -1 键值对（40多亿）。

常用命令：hget、hset、hgetall等。

应用场景：存储一些结构化的数据，比如用户的昵称、年龄、性别、积分等，存储一个用户信息对象数据。

```bash
hset person name bingo
hset person age 20
hset person id 1
hget person name
(person = {
  "name": "bingo",
  "age": 20,
  "id": 1
})
```



### Lists

Redis 列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素到列表的头部（左边）或者尾部（右边）。

list类型经常会被用于消息队列的服务，以完成多程序之间的消息交换。

常用命令：lpush、rpush、lpop、rpop、lrange等。

列表最多可存储 232 - 1 元素 (4294967295, 每个列表可存储40多亿)。

应用场景：

+ 比如可以通过 list 存储一些列表型的数据结构，类似粉丝列表、文章的评论列表之类的东西。

+ 比如可以通过 lrange 命令，读取某个闭区间内的元素，可以基于 list 实现分页查询，这个是很棒的一个功能，基于 Redis 实现简单的高性能分页，可以做类似微博那种下拉不断分页的东西，性能高，就一页一页走。

  ```
  # 0开始位置，-1结束位置，结束位置为-1时，表示列表的最后一个位置，即查看所有。
  lrange mylist 0 -1
  ```

+ 比如可以搞个简单的消息队列，从 list 头怼进去，从 list 尾巴那里弄出来。

  ```
  lpush mylist 1
  lpush mylist 2
  lpush mylist 3 4 5
  
  # 1
  rpop mylist
  ```

  



### Sets

Redis的Set是string类型的无序集合(会自动去重。)。和列表一样，在执行插入和删除和判断是否存在某元素时，效率是很高的。集合最大的优势在于可以进行交集并集差集操作。Set可包含的最大元素数量是4294967295。
集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是O(1)。

应用场景：

+ 可以玩儿交集、并集、差集的操作，比如交集吧，可以把两个人的粉丝列表整一个交集，看看俩人的共同好友是谁，把两个大 V 的粉丝都放在两个 set 中，对两个 set 做交集：

  ```
  #-------操作一个set-------
  # 添加元素
  sadd mySet 1
  # 查看全部元素
  smembers mySet
  # 判断是否包含某个值
  sismember mySet 3
  # 删除某个/些元素
  srem mySet 1
  srem mySet 2 4
  # 查看元素个数
  scard mySet
  # 随机删除一个元素
  spop mySet
  #-------操作多个set-------
  # 将一个set的元素移动到另外一个set
  smove yourSet mySet 2
  # 求两set的交集
  sinter yourSet mySet
  # 求两set的并集
  sunion yourSet mySet
  # 求在yourSet中而不在mySet中的元素
  sdiff yourSet mySet
  ```

  

+ 利用唯一性，可以统计访问网站的所有独立IP。

+ 好友推荐的时候根据tag求交集，大于某个threshold（临界值的）就可以推荐。

常用命令：sadd、spop、smembers、sunion等。

集合中最大的成员数为 232 - 1(4294967295, 每个集合可存储40多亿个成员)。



### zsets（Sorted Sets）

Redis zset 和 set 一样也是string类型元素的集合,且不允许重复的成员。

不同的是每个元素都会关联一个double类型的分数。redis正是通过分数来为集合中的成员进行从小到大的排序。

zset的成员是唯一的,但分数(score)却可以重复。

sorted set是插入有序的，即自动排序。

常用命令：zadd、zrange、zrem、zcard等。

当你需要一个有序的并且不重复的集合列表时，那么可以选择sorted set数据结构。

写入数据时候带一个分数值，将自动按照分数进行排序。

```
zadd board 85 zhangsan
zadd board 72 lisi
zadd board 96 wangwu
zadd board 63 zhaoliu
# 获取排名前三的用户（默认是升序，所以需要 rev 改为降序）
zrevrange board 0 3
# 获取某用户的排名
zrank board zhaoliu
```

应用举例：

+ 例如存储全班同学的成绩，其集合value可以是同学的学号，而score就可以是成绩。
+ 排行榜应用，根据得分列出topN的用户等。



## 分布式锁实现

```xml
<dependency>
   <groupId>org.redisson</groupId>
   <artifactId>redisson</artifactId>
   <version>3.16.3</version>
</dependency>  
```

```java
// 1. Create config object
Config config = new Config();
config.useClusterServers()
       // use "rediss://" for SSL connection
      .addNodeAddress("redis://127.0.0.1:7181");

// or read config from file
config = Config.fromYAML(new File("config-file.yaml")); 
// 2. Create Redisson instance

// Sync and Async API
RedissonClient redisson = Redisson.create(config);

// Reactive API
RedissonReactiveClient redissonReactive = redisson.reactive();

// RxJava3 API
RedissonRxClient redissonRx = redisson.rxJava();
// 3. Get Redis based implementation of java.util.concurrent.ConcurrentMap
RMap<MyKey, MyValue> map = redisson.getMap("myMap");

RMapReactive<MyKey, MyValue> mapReactive = redissonReactive.getMap("myMap");

RMapRx<MyKey, MyValue> mapRx = redissonRx.getMap("myMap");
// 4. Get Redis based implementation of java.util.concurrent.locks.Lock
RLock lock = redisson.getLock("myLock");

RLockReactive lockReactive = redissonReactive.getLock("myLock");

RLockRx lockRx = redissonRx.getLock("myLock");
// 4. Get Redis based implementation of java.util.concurrent.ExecutorService
RExecutorService executor = redisson.getExecutorService("myExecutorService");

// over 50 Redis based Java objects and services ...
```

## 订阅发布模式

可以通过订阅发布模式做一个很简单的即时通信组件

```
#A监听A频道
subscribe a
#B给A发消息
publish a helloworld
#此时A可以收到helloworld的消息
```



## 变相实现给list元素设置有效时间

结合有序集合，加入到有序集合的每个项，我们都将它的score设置为一个时间戳，这个时间戳代表它的过期时间。然后，我们加入一个定时任务，定时移除那些过期的数据即可。
