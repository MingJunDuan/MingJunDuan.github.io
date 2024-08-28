---
layout: post
title: "Redis-bitmap源码分析"
author: "Inela"
---

Redis源码5.0版本

Redis的bitmap用于存储位图数据。它可以表示大量的位（bit）并对位进行操作，具有高效的存储和计算性能，可以用在用户在线状态跟踪(离线在线)、消息推送和通知、用户活跃度统计、布隆过滤器等场景。

来分析下setbit、getbit、bitcount命令是操作bitmap的，源码解析如下。


- setbit命令下标值设置0或者1，其返回值是旧值，bitmap中值只能是0或1
- getbit获取下标值是0还是1
- bitcount是下标为1的个数
- 还有其他命令，bitop：对多个位数组进行按位与、或、异或运算

```bash
#设置100000下标对应的为1
127.0.0.1:6379> setbit mybitmap 100000 1
(integer) 0
127.0.0.1:6379> getbit mybitmap 100000
(integer) 1
#下标参数只能是正整数
127.0.0.1:6379> setbit mybitmap aa
(error) ERR wrong number of arguments for 'setbit' command
127.0.0.1:6379> setbit mybitmap -1
(error) ERR wrong number of arguments for 'setbit' command
127.0.0.1:6379> getbit mybitmap aa
(error) ERR bit offset is not an integer or out of range
#mybitmap这个bitmap中下标值为1的个数
127.0.0.1:6379> bitcount mybitmap
(integer) 3
```

# setbit源码解析

源码如下，第0个参数setbit，第1个参数是key，第2个参数是下标，第3个参数是值(0或1)

- 解析第2个参数，即下标；解析第3个参数，即value，value只能是0或者1，否则报错
- lookupStringForBitCommand方法根据key查询SDS对象，如果key不存在则创建
- bitoffset >> 3计算出buf[index]的index，可以知道是哪个buf数组下标，之后计算新值然后更新buf[index]

```c
void setbitCommand(client *c) {
    robj *o;
    char *err = "bit is not an integer or out of range";
    uint64_t bitoffset;
    ssize_t byte, bit;
    int byteval, bitval;
    long on;
    //解析bitoffset参数, 下标为2的参数
    if (getBitOffsetFromArgument(c,c->argv[2],&bitoffset,0,0) != C_OK)
        return;
    //解析value, 下标为3的参数
    if (getLongFromObjectOrReply(c,c->argv[3],&on,err) != C_OK)
        return;
    //value只能是0或者1
    /* Bits can only be set or cleared... */
    if (on & ~1) {
        addReplyError(c,err);
        return;
    }
		//key查询SDS对象（会自动扩容）
    if ((o = lookupStringForBitCommand(c,bitoffset)) == NULL) return;

    /* 获取当前值 */
    //计算是哪个buf[byte]
    byte = bitoffset >> 3;
    //获取buf[byte]的值，是个byte
    byteval = ((uint8_t*)o->ptr)[byte];
    bit = 7 - (bitoffset & 0x7);
    bitval = byteval & (1 << bit);

    /* 更新新值，Update byte with new bit value and return original value */
    byteval &= ~(1 << bit);
    byteval |= ((on & 0x1) << bit);
    ((uint8_t*)o->ptr)[byte] = byteval;
  	//返回旧值
    signalModifiedKey(c->db,c->argv[1]);
    notifyKeyspaceEvent(NOTIFY_STRING,"setbit",c->argv[1],c->db->id);
    server.dirty++;
    addReply(c, bitval ? shared.cone : shared.czero);
}
```

以**setbit 10 1**命令为例如下：

![setbit命令](https://github.com/MingJunDuan/mingjunduan.github.io/raw/main/images/mjduan/2024-07-21/setbit命令.png)

寻找 key是否存在：

- 首先调用lookupKeyWrite获取key；
- 如果不存在，则创建对象(SDS，string类型，底层是byte[])，长度刚好能够存储maxbit，并设置到db.dict中。所以在使用时不用管bitmap key是否存在，不存在则会自动创建的；
- 如果存在则返回

```c
robj *lookupStringForBitCommand(client *c, uint64_t maxbit) {
    size_t byte = maxbit >> 3;
    robj *o = lookupKeyWrite(c->db,c->argv[1]);

    if (o == NULL) {
        o = createObject(OBJ_STRING,sdsnewlen(NULL, byte+1));
        dbAdd(c->db,c->argv[1],o);
    } else {
        if (checkType(c,o,OBJ_STRING)) return NULL;
        o = dbUnshareStringValue(c->db,c->argv[1],o);
        o->ptr = sdsgrowzero(o->ptr,byte+1);
    }
    return o;
}
```

- 从redis的db.dict中获取key对应的dictEntry
- 获取对应的value返回，在返回前会更新value的access time，用于maxmemory policy

```c
robj *lookupKey(redisDb *db, robj *key, int flags) {
    dictEntry *de = dictFind(db->dict,key->ptr);
    if (de) {
        robj *val = dictGetVal(de);

        /* Update the access time for the ageing algorithm.
         * Don't do it if we have a saving child, as this will trigger
         * a copy on write madness. */
        if (server.rdb_child_pid == -1 &&
            server.aof_child_pid == -1 &&
            !(flags & LOOKUP_NOTOUCH))
        {
            if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
                updateLFU(val);
            } else {
                val->lru = LRU_CLOCK();
            }
        }
        return val;
    } else {
        return NULL;
    }
}
```

key从Redis的dict字典中获取：

- 如果dict处于rehash中，则按rehash的逻辑处理
- 否则计算key的hash值，获取dict.ht[0].table[hash]、dict.ht[1].table[hash]链表，之后遍历链表获取对应的key

```c
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he;
    uint64_t h, idx, table;

    if (d->ht[0].used + d->ht[1].used == 0) return NULL; /* dict is empty */
    if (dictIsRehashing(d)) _dictRehashStep(d);
    h = dictHashKey(d, key);
    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask;
        he = d->ht[table].table[idx];
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key))
                return he;
            he = he->next;
        }
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL;
}
```



# getbit源码解析

​	getbit操作类似setbit，只是getbit获取对应下标的值，而不是设置值；



# bitcount源码解析

- bitcount采用SWAR算法，其实还采用了bitsinbyte预生成数组去减少计算开销；
- 不是采用挨个遍历的算法

```c
void bitcountCommand(client *c) {
    robj *o;
    long start, end, strlen;
    unsigned char *p;
    char llbuf[LONG_STR_SIZE];

    /* Lookup, check for type, and return 0 for non existing keys. */
    if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.czero)) == NULL ||
        checkType(c,o,OBJ_STRING)) return;
    p = getObjectReadOnlyString(o,&strlen,llbuf);

    /* Parse start/end range if any. */
    if (c->argc == 4) {
        if (getLongFromObjectOrReply(c,c->argv[2],&start,NULL) != C_OK)
            return;
        if (getLongFromObjectOrReply(c,c->argv[3],&end,NULL) != C_OK)
            return;
        /* Convert negative indexes */
        if (start < 0 && end < 0 && start > end) {
            addReply(c,shared.czero);
            return;
        }
        if (start < 0) start = strlen+start;
        if (end < 0) end = strlen+end;
        if (start < 0) start = 0;
        if (end < 0) end = 0;
        if (end >= strlen) end = strlen-1;
    } else if (c->argc == 2) {
        /* The whole string. */
        start = 0;
        end = strlen-1;
    } else {
        /* Syntax error. */
        addReply(c,shared.syntaxerr);
        return;
    }

    /* Precondition: end >= 0 && end < strlen, so the only condition where
     * zero can be returned is: start > end. */
    if (start > end) {
        addReply(c,shared.czero);
    } else {
        long bytes = end-start+1;

        addReplyLongLong(c,redisPopcount(p+start,bytes));
    }
}
```

redisPopcount实现如下：

```c
long long redisPopcount(void *s, long count) {
    long long bits = 0;
    unsigned char *p = s;
    uint32_t *p4;
    static const unsigned char bitsinbyte[256] = {0,1,1,2,1,2,2,3,1,2,2,3,2,3,3,4,1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,4,5,5,6,5,6,6,7,5,6,6,7,6,7,7,8};

    /* Count initial bytes not aligned to 32 bit. */
    while((unsigned long)p & 3 && count) {
        bits += bitsinbyte[*p++];
        count--;
    }

    /* Count bits 28 bytes at a time */
    p4 = (uint32_t*)p;
    while(count>=28) {
        uint32_t aux1, aux2, aux3, aux4, aux5, aux6, aux7;

        aux1 = *p4++;
        aux2 = *p4++;
        aux3 = *p4++;
        aux4 = *p4++;
        aux5 = *p4++;
        aux6 = *p4++;
        aux7 = *p4++;
        count -= 28;

        aux1 = aux1 - ((aux1 >> 1) & 0x55555555);
        aux1 = (aux1 & 0x33333333) + ((aux1 >> 2) & 0x33333333);
        aux2 = aux2 - ((aux2 >> 1) & 0x55555555);
        aux2 = (aux2 & 0x33333333) + ((aux2 >> 2) & 0x33333333);
        aux3 = aux3 - ((aux3 >> 1) & 0x55555555);
        aux3 = (aux3 & 0x33333333) + ((aux3 >> 2) & 0x33333333);
        aux4 = aux4 - ((aux4 >> 1) & 0x55555555);
        aux4 = (aux4 & 0x33333333) + ((aux4 >> 2) & 0x33333333);
        aux5 = aux5 - ((aux5 >> 1) & 0x55555555);
        aux5 = (aux5 & 0x33333333) + ((aux5 >> 2) & 0x33333333);
        aux6 = aux6 - ((aux6 >> 1) & 0x55555555);
        aux6 = (aux6 & 0x33333333) + ((aux6 >> 2) & 0x33333333);
        aux7 = aux7 - ((aux7 >> 1) & 0x55555555);
        aux7 = (aux7 & 0x33333333) + ((aux7 >> 2) & 0x33333333);
        bits += ((((aux1 + (aux1 >> 4)) & 0x0F0F0F0F) +
                    ((aux2 + (aux2 >> 4)) & 0x0F0F0F0F) +
                    ((aux3 + (aux3 >> 4)) & 0x0F0F0F0F) +
                    ((aux4 + (aux4 >> 4)) & 0x0F0F0F0F) +
                    ((aux5 + (aux5 >> 4)) & 0x0F0F0F0F) +
                    ((aux6 + (aux6 >> 4)) & 0x0F0F0F0F) +
                    ((aux7 + (aux7 >> 4)) & 0x0F0F0F0F))* 0x01010101) >> 24;
    }
    /* Count the remaining bytes. */
    p = (unsigned char*)p4;
    while(count--) bits += bitsinbyte[*p++];
    return bits;
}
```



参考：
1. https://www.51cto.com/article/702541.html

