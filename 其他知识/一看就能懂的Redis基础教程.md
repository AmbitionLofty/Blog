

##  一看就能懂的Redis基础教程

@(Java)


## Redis 简介
Redis 是完全开源免费的，遵守BSD协议，是一个高性能的key-value数据库。
Redis 与其他 key - value 缓存产品有以下三个特点：

 - Redis支持数据的持久化，**可以将内存中的数据保存在磁盘中，重启的时候可以再次加载进行使用**。
 - Redis不仅仅支持简单的key-value类型的数据，**同时还提供list，set，zset，hash等数据结构的存储**。
 - Redis**支持数据的备份**，即master-slave模式的数据备份。

## Redis 优势

 - **性能极高** – Redis能读的速度是110000次/s,写的速度是81000次/s 。
 - **丰富的数据类型** – Redis支持二进制案例的 Strings, Lists, Hashes, Sets 及 Ordered Sets
   数据类型操作。
 - **原子** – Redis的所有操作都是原子性的，同时Redis还支持对几个操作全并后的原子性执行。
 - **丰富的特性** – Redis还支持 publish/subscribe, 通知, key 过期等等特性。

### Redis与其他key-value存储有什么不同？

 - Redis有着**更为复杂的数据结构并且提供对他们的原子性操作**，这是一个不同于其他数据库的进化路径。Redis的数据类型都是基于基本数据结构的**同时对程序员透明**，无需进行额外的抽象。
 - Redis**运行在内存中但是可以持久化到磁盘**，所以在对不同数据集进行高速读写时需要权衡内存，因为数据量不能大于硬件内存。在内存数据库方面的另一个优点是，相比在磁盘上相同的复杂的数据结构，在内存中操作起来非常简单，这样Redis可以做很多内部复杂性很强的事情。同时，在磁盘格式方面他们是紧凑的以追加的方式产生的，因为他们并不需要进行随机访问。


## Redis 数据类型

Redis支持五种数据类型：**string（字符串），hash（哈希），list（列表），set（集合）及zset(sorted set：有序集合)**。
### String（字符串）
string是redis最基本的类型，你可以理解成与Memcached一模一样的类型，一个key对应一个value。
string类型是二进制安全的。意思是redis的string可以包含任何数据。比如jpg图片或者序列化的对象 。
string类型是Redis最基本的数据类型，一个键最大能存储512MB。

#### 实例

```
redis 127.0.0.1:6379> SET name "runoob"
OK
redis 127.0.0.1:6379> GET name
"runoob"
```

在以上实例中我们使用了 Redis 的 SET 和 GET 命令。键为 name，对应的值为 runoob。
注意：一个键最大能存储512MB。
### Hash（哈希）
Redis hash 是一个键名对集合。
Redis hash是一个string类型的field和value的映射表，hash特别适合用于存储对象。
#### 实例

```
127.0.0.1:6379> HMSET user:1 username runoob password runoob points 200
OK
127.0.0.1:6379> HGETALL user:1
1) "username"
2) "runoob"
3) "password"
4) "runoob"
5) "points"
6) "200"
```
以上实例中 hash 数据类型存储了包含用户脚本信息的用户对象。 实例中我们使用了 Redis HMSET, HGETALL 命令，user:1 为键值。
每个 hash 可以存储 2的32 -1 键值对（40多亿）。

### List（列表）
Redis 列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素到列表的头部（左边）或者尾部（右边）。


#### 实例
```
redis 127.0.0.1:6379> lpush runoob redis
(integer) 1
redis 127.0.0.1:6379> lpush runoob mongodb
(integer) 2
redis 127.0.0.1:6379> lpush runoob rabitmq
(integer) 3
redis 127.0.0.1:6379> lrange runoob 0 10
1) "rabitmq"
2) "mongodb"
3) "redis"
redis 127.0.0.1:6379>
```
列表最多可存储 232 - 1 元素 (4294967295, 每个列表可存储40多亿)。


### Set（集合）

Redis的Set是string类型的无序集合。
集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是O(1)。


#### sadd 命令
添加一个string元素到,key对应的set集合中，成功返回1,如果元素已经在集合中返回0,key对应的set不存在返回错误。

```
sadd key member
```

#### 实例

```
redis 127.0.0.1:6379> sadd runoob redis
(integer) 1
redis 127.0.0.1:6379> sadd runoob mongodb
(integer) 1
redis 127.0.0.1:6379> sadd runoob rabitmq
(integer) 1
redis 127.0.0.1:6379> sadd runoob rabitmq
(integer) 0
redis 127.0.0.1:6379> smembers runoob

1) "rabitmq"
2) "mongodb"
3) "redis"
```

注意：以上实例中 rabitmq 添加了两次，但根据集合内元素的唯一性，第二次插入的元素将被忽略。
集合中最大的成员数为 232 - 1(4294967295, 每个集合可存储40多亿个成员)。


### zset(sorted set：有序集合)

Redis zset 和 set 一样也是string类型元素的集合,且不允许重复的成员。
不同的是每个元素都会关联一个double类型的分数。redis正是通过分数来为集合中的成员进行从小到大的排序。
zset的成员是唯一的,但分数(score)却可以重复。

#### zadd 命令
添加元素到集合，元素在集合中存在则更新对应score

```
zadd key score member 
```

#### 实例


```
redis 127.0.0.1:6379> zadd runoob 0 redis
(integer) 1
redis 127.0.0.1:6379> zadd runoob 0 mongodb
(integer) 1
redis 127.0.0.1:6379> zadd runoob 0 rabitmq
(integer) 1
redis 127.0.0.1:6379> zadd runoob 0 rabitmq
(integer) 0
redis 127.0.0.1:6379> ZRANGEBYSCORE runoob 0 1000

1) "redis"
2) "mongodb"
3) "rabitmq"
```



OVER