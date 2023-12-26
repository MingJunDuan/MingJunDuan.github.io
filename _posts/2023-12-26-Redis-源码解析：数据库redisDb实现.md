---
layout: post
title: "Redis-源码解析：数据库redisDb实现"
author: "Inela"
---

# 服务器中的数据库
 
 Redis 服务器将绝大部分的信息都保存在 `server.h/redisServer`。redis 的数据是保存在 `redisServer` 中的 `redisDb` 结构中。
 
 ```c
 struct redisServer {
     // ...
     redisDb *db; // 数据库列表
     // ...
     int dbnum;   // 数据库数量
     // ...
 }
 ```
 
 - `db` 中每个redisDb结构代表一个数据库。
 - 在初始化服务器时，程序会根据服务器状态的 `dbnum` 属性来决定应该创建多少个数据库。
 - `dbnum` 属性的值由服务器配置的 `database` 选项决定，默认情况下，该选项的值为16，所以Redis服务器默认会创建16个数据库。
 
 ![redisServer数据结构](https://github.com/MingJunDuan/mingjunduan.github.io/raw/main/images/mjduan/2023-12-26/redisServer数据结构.png)
 
 
 
# 数据库键空间
 
 Redis 是一个键值对数据库服务器，服务器中的每个数据库都由一个 `server.h/redisDb` 结构表示.
 其中，`redisDb` 的 `dict` 字典属性保存了数据库中的所有键值对，我们将这个字典称为键空间（key space）：
 
 ```c
 /* Redis database representation. There are multiple databases identified
  * by integers from 0 (the default database) up to the max configured
  * database. The database number is the 'id' field in the structure. */
 //Redis数据库结构，通过ID来识别多个数据库
 typedef struct redisDb {
     // 当前数据口的所有键值对空间
     dict *dict;                 /* The keyspace for this DB */
     // 存放已设置exptime的key的集合
     dict *expires;              /* Timeout of keys with a timeout set */
     ....
     ....
     // 数据库id
     int id;                     /* Database ID */
     // 数据库的平均TTL
     long long avg_ttl;          /* Average TTL, just for stats */
     unsigned long expires_cursor; /* Cursor of the active expire cycle. */
     list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
     clusterSlotToKeyMapping *slots_to_keys; /* Array of slots to keys. Only used in cluster mode (db 0). */
 } redisDb;
 ```
 
 dict 中的数据跟我们平常操作的键值对是一一对应的：
 
 - dict 的 key 就是数据库中的 key，字符串类型
 - dict 的 值 就是数据库中的 值，这个值可以是 `string`、`hash`、`zset`、`set`、`list` 中的任何一种
 
 ### 示例
 
 如果我们在数据库中，执行以下命令：
 
 ```shell
 redis > SET str_key str_value
 OK
 redis > RPUSH list_key a b c
 (integer) 3
 ```
 
 新添加的两个 key 的结构如下图所示：
 
 ![redisDB数据结构](https://github.com/MingJunDuan/mingjunduan.github.io/raw/main/images/mjduan/2023-12-26/redisDB数据结构.png)
 
 从上面的示例图可以很清晰地知道 Redis 数据是如何组织的，增删改查也就是对 dict 的操作而已
 
# Key 的过期时间
 
 ### 1. 数据结构
 
 redisDb 中的 `expires` 属性保存了存放已设置exptime的key的集合的过期时间，我们姑且就称它为**过期字典**吧。
 
 - 过期字典中的键，是一个指针，指向了真实数据的 `key`，不会浪费空间多保存一次
 - 过期字典中的值，存的是具体的过期时间点，精确到毫秒的时间戳
 
 ```c
 /* Redis database representation. There are multiple databases identified
  * by integers from 0 (the default database) up to the max configured
  * database. The database number is the 'id' field in the structure. */
 //Redis数据库结构，通过ID来识别多个数据库
 typedef struct redisDb {
     // 当前数据口的所有键值对空间
     dict *dict;                 /* The keyspace for this DB */
     // 存放已设置exptime的key的集合
     dict *expires;              /* Timeout of keys with a timeout set */
     ....
     ....
     // 数据库id
     int id;                     /* Database ID */
     // 数据库的平均TTL
     long long avg_ttl;          /* Average TTL, just for stats */
     unsigned long expires_cursor; /* Cursor of the active expire cycle. */
     list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
     clusterSlotToKeyMapping *slots_to_keys; /* Array of slots to keys. Only used in cluster mode (db 0). */
 } redisDb;
 ```
 
 命令`TTL`、`PTTL` 都是去查这个过期字典的过期时间，然后减去当前时间，得到的就是剩余的时间啦。
 
 **那么expires中entry的key和value代表什么？**
 
 - key：某个键对象
 - value：long long类型的整数，表示key的过期时间
 
 ![redisDB的key过期时间](https://github.com/MingJunDuan/mingjunduan.github.io/raw/main/images/mjduan/2023-12-26/redisDB的key过期时间.png)
 
 ### 2. 过期 key 的删除策略
 
 一个 key 过期时间到了之后，是如何进行删除的呢？Redis 使用了一下两种策略：惰性删除、定期删除
 
 #### 惰性删除[](https://www.cnblogs.com/chenchuxin/p/14187898.html#惰性删除)
 
 惰性删除策略指的是：key 在过期之后，没有立即删除，而是在读写 key 的时候，才对过期的 key 进行删除。
 代码实现在 `db.c/expireIfNeeded` 方法中。所有 key 的读写之前，都会先调用 `expireIfNeeded` 对 key 进行检查，如果已过期，则删除。
 
 ```
 int expireIfNeeded(redisDb *db, robj *key, int flags) {
     if (server.lazy_expire_disabled) return 0;
     //看key是否过期了
     if (!keyIsExpired(db,key)) return 0;
     ...
     deleteExpiredKeyAndPropagate(db,key);
     ...
     return 1;
 }
 
 int keyIsExpired(redisDb *db, robj *key) {
     /* Don't expire anything while loading. It will be done later. */
     if (server.loading) return 0;
     
     //获取key对应的过期时间，存储的是时间戳
     mstime_t when = getExpire(db,key);
     mstime_t now;
 
     if (when < 0) return 0; /* No expire for this key */
 
     //获取系统的当前时间，时间戳，是毫秒级的long类型数据
     now = commandTimeSnapshot();
 
     /* The key expired if the current (virtual or real) time is greater
      * than the expire time of the key. */
     //如果过了过期时间点
     return now > when;
 }
 ```
 
 
 
 #### 定期删除
 
 定期删除策略指的是：Redis 每隔一段时间，随机从数据库中取出一定量的 key 进行检查，如果已过期，则进行删除。
 代码实现在 `expire.c/activeExpireCycle` 方法中。
 
 删除的步骤：
 
 1. 从过期字典中随机 20 个 key
 2. 删除这 20 个 key 中已经过期的 key
 3. 如果过期的 key 比率超过 1/4，那就重复步骤 1
 
 为什么只是随机挑 一些 key 呢？因为如果把所有 key 都遍历一遍，那这个性能肯定是不能接受的！
 
 
 
# 总结
 
 redis中所有key都存储在redisDB的dict字典中，增删改查也就是对 dict 的操作而已，而对于设置了expire过期时间的key来说，在expire字段(字典类型)中也存储了该字段，value是过期时间(时间戳类型)。
 
 过期key的删除时惰性删除结合定期删除，在操作redis key时，会先判断该key是否已过期，如果过期了则进行删除。
 
 
 
 参考：
 
 1.https://www.wzl.fyi/2023/07/29/1008/
 
 2.https://www.cnblogs.com/chenchuxin/p/14187898.html

