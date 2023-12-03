---
layout: post
title: "Redis-Dict字典实现"
author: "Inela"
---



如下是dict字典的数据结构

1. ht[2]对应俩个hash，主要使用ht[0]存储数据，ht[1]用于进行扩容
2. rehashidx一般是-1表示当前没有在进行扩容，rehashidx如果是2，则表示当前在进行ht[0]到ht[1]的数据迁移，正在迁移
3. ht[0]中下标是2的槽位对应的链表
4. Ht[0]中的used表示ht[0]的一位数组槽位上多少个是有值的，即不为null的槽位个数



#dict.h

```
//字典
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;

//Hash哈希表
/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

//迭代器
/* If safe is set to 1 this is a safe iterator, that means, you can call
 * dictAdd, dictFind, and other functions against the dictionary even while
 * iterating. Otherwise it is a non safe iterator, and only dictNext()
 * should be called while iterating. */
typedef struct dictIterator {
    dict *d;
    long index;
    int table, safe;
    dictEntry *entry, *nextEntry;
    /* unsafe iterator fingerprint for misuse detection. */
    unsigned long long fingerprint;
} dictIterator;

```



![dict字典的数据结构](https://github.com/MingJunDuan/mingjunduan.github.io/raw/main/images/mjduan/2023-11-30/dict字典的数据结构.png)



#bitops.c

```
/* GETBIT key offset */
void getbitCommand(client *c) {
    robj *o;
    char llbuf[32];
    uint64_t bitoffset;
    size_t byte, bit;
    size_t bitval = 0;

    if (getBitOffsetFromArgument(c,c->argv[2],&bitoffset,0,0) != C_OK)
        return;
		//从redis的db中获取key的value
    if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.czero)) == NULL ||
        checkType(c,o,OBJ_STRING)) return;

    byte = bitoffset >> 3;
    bit = 7 - (bitoffset & 0x7);
    if (sdsEncodedObject(o)) {
        if (byte < sdslen(o->ptr))
            bitval = ((uint8_t*)o->ptr)[byte] & (1 << bit);
    } else {
        if (byte < (size_t)ll2string(llbuf,sizeof(llbuf),(long)o->ptr))
            bitval = llbuf[byte] & (1 << bit);
    }

    addReply(c, bitval ? shared.cone : shared.czero);
}
```



#db.c

```
robj *lookupKeyReadOrReply(client *c, robj *key, robj *reply) {
		//从db中查找
    robj *o = lookupKeyRead(c->db, key);
    if (!o) addReply(c,reply);
    return o;
}
```

​	dict字段渐进式扩容，会在某次请求中将这个hash

#db.c

```
robj *lookupKey(redisDb *db, robj *key, int flags) {
		//每个db里面底层是一个dict，从dict中查找key
    dictEntry *de = dictFind(db->dict,key->ptr);
    if (de) {
        
        robj *val = dictGetVal(de);

        /* Update the access time for the ageing algorithm.
         * Don't do it if we have a saving child, as this will trigger
         * a copy on write madness. */
        if (!hasActiveChildProcess() && !(flags & LOOKUP_NOTOUCH)){
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



#dict.c

```
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he;
    uint64_t h, idx, table;
		//如果底层的dict是空的，则说明redis中没有写入过数据
    if (dictSize(d) == 0) return NULL; /* dict is empty */
    //如果当前dict处于扩容中，则进行渐进式迁移，见_dictRehashStep中
    if (dictIsRehashing(d)) _dictRehashStep(d);
    h = dictHashKey(d, key);
    //查询ht[0]，如果处于扩容中，则还会查询ht[1]
    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask;
        //获取到hash表的槽位bucket头部元素
        he = d->ht[table].table[idx];
        //遍历链表
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



#dict.h

```
//dict的rehashidx不是-1则表示当前处于扩容中
#define dictIsRehashing(d) ((d)->rehashidx != -1)
```



#dict.c

```
static void _dictRehashStep(dict *d) {
		//迭代器值是0表示，当前没有迭代器在进行遍历，可以进行dict扩容
    if (d->iterators == 0) dictRehash(d,1);
}
```



#dict.c

```
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    unsigned long s0 = d->ht[0].size;
    unsigned long s1 = d->ht[1].size;
    if (dict_can_resize == DICT_RESIZE_FORBID || !dictIsRehashing(d)) return 0;
    if (dict_can_resize == DICT_RESIZE_AVOID && 
        ((s1 > s0 && s1 / s0 < dict_force_resize_ratio) ||
         (s1 < s0 && s0 / s1 < dict_force_resize_ratio)))
    {
        return 0;
    }
		//dict中的ht[0].used!=0表示ht[0]中还有元素，需要进行转移
		//会在一次客户端请求中将ht[0]中的m某个槽位上的链表元素全部迁移到ht[1]上
    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        //跳过bucket槽位是null的，因为bucket为null不需要转移元素
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        //获取要转移的槽位
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            uint64_t h;
						//遍历该槽位对应的链表
            nextde = de->next;
            /* Get the index in the new hash table */
            //将元素转移到ht[1]中，先计算ht[1]中的index
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            //头插入法
            d->ht[1].table[h] = de;
            //将ht[0]、ht[1]的used值分别减、加1，used值表示的是槽位bucket不为null的个数
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        //将ht[0]的已转移的槽位bucket置为null
        d->ht[0].table[d->rehashidx] = NULL;
        //dict的rehash指的是ht[0]/ht[1]中槽位bucket的下标
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */
    if (d->ht[0].used == 0) {
        //释放dict的ht[0]
        zfree(d->ht[0].table);
        //将ht[1]赋值给ht[0]，引用赋值
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        //将dict的rehashidx设置为-1，表示没有在扩容
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```



参考：

1. https://www.cnblogs.com/chenchuxin/p/14191156.html

   