---
layout: post
title: "Redis-源码解析：scan源码实现"
author: "Inela"
---

​	由于历史原因，有个项目遗留了个上百G的Redis，通过使用scan最获取后一次访问时间，如果最后一次访问时间超3个月则删除，这样清理和上百G的Redis存储，来看下scan是如何实现的。

​	
- keys命令会一次性返回所有的key，如果db中key很多，很阻塞业务请求，生产环境是禁止使用的；
- scan 命令是一个基于游标的迭代器，每次被调用之后， 都会向用户返回一个新的游标， 用户在下次迭代时需要使用这个新游标作为 scan 命令的游标参数， 以此来延续之前的迭代过程；

```
/* The SCAN command completely relies on scanGenericCommand. */
void scanCommand(client *c) {
    unsigned long cursor;
    if (parseScanCursorOrReply(c,c->argv[1],&cursor) == C_ERR) return;
    //传入到scanGenericCommand的是NULL
    scanGenericCommand(c,NULL,cursor);
}
```

​	参数o是null

```
void scanGenericCommand(client *c, robj *o, unsigned long cursor) {
    int i, j;
    list *keys = listCreate();
    listNode *node, *nextnode;
    long count = 10;
    sds pat = NULL;
    int patlen = 0, use_pattern = 0;
    dict *ht;

    /* Set i to the first option argument. The previous one is the cursor. */
    i = (o == NULL) ? 2 : 3; /* Skip the key argument if needed. */

    /* Step 1: Parse options. */
    ...

    /* Handle the case of a hash table. */
    ht = NULL;
    if (o == NULL) {
        //访问字典
        ht = c->db->dict;
    ...

    if (ht) {
        void *privdata[2];
        /* We set the max number of iterations to ten times the specified
         * COUNT, so if the hash table is in a pathological state (very
         * sparsely populated) we avoid to block too much time at the cost
         * of returning no or very few elements. */
        long maxiterations = count*10;

        /* We pass two pointers to the callback: the list to which it will
         * add new elements, and the object containing the dictionary so that
         * it is possible to fetch more data in a type-dependent way. */
        privdata[0] = keys;
        privdata[1] = o;
        do {
        		//访问字典中的key
            cursor = dictScan(ht, cursor, scanCallback, NULL, privdata);
        } while (cursor &&
              maxiterations-- &&
              listLength(keys) < (unsigned long)count);
    } else if (o->type == OBJ_SET) {
      ...

    /* Step 3: Filter elements. */
    //过滤

    /* Step 4: Reply to the client. */
    addReplyMultiBulkLen(c, 2);
    addReplyBulkLongLong(c,cursor);

    addReplyMultiBulkLen(c, listLength(keys));
    while ((node = listFirst(keys)) != NULL) {
        robj *kobj = listNodeValue(node);
        addReplyBulk(c, kobj);
        decrRefCount(kobj);
        listDelNode(keys, node);
    }

cleanup:
    listSetFreeMethod(keys,decrRefCountVoid);
    listRelease(keys);
}
```



​	如下调用dict的方法进行遍历key，考虑rehash的情况

​	#dict.c

```
unsigned long dictScan(dict *d,
                       unsigned long v,
                       dictScanFunction *fn,
                       dictScanBucketFunction* bucketfn,
                       void *privdata)
{
    dictht *t0, *t1;
    const dictEntry *de, *next;
    unsigned long m0, m1;

    if (dictSize(d) == 0) return 0;

    if (!dictIsRehashing(d)) {
    		//没有在rehash则遍历hash[0]
        t0 = &(d->ht[0]);
        m0 = t0->sizemask;

        /* Emit entries at cursor */
        if (bucketfn) bucketfn(privdata, &t0->table[v & m0]);
        //从第v个hash槽开始遍历
        de = t0->table[v & m0];
        while (de) {
            next = de->next;
            fn(privdata, de);
            de = next;
        }

        /* Set unmasked bits so incrementing the reversed cursor
         * operates on the masked bits */
        v |= ~m0;

        /* Increment the reverse cursor */
        v = rev(v);
        v++;
        v = rev(v);

    } else {
    		//表示正在hash
        t0 = &d->ht[0];
        t1 = &d->ht[1];

        /* Make sure t0 is the smaller and t1 is the bigger table */
        if (t0->size > t1->size) {
            t0 = &d->ht[1];
            t1 = &d->ht[0];
        }

        m0 = t0->sizemask;
        m1 = t1->sizemask;

        /* Emit entries at cursor */
        if (bucketfn) bucketfn(privdata, &t0->table[v & m0]);
        //先遍历元素多的hash
        de = t0->table[v & m0];
        while (de) {
            next = de->next;
            fn(privdata, de);
            de = next;
        }

        /* Iterate over indices in larger table that are the expansion
         * of the index pointed to by the cursor in the smaller table */
        do {
            /* Emit entries at cursor */
            if (bucketfn) bucketfn(privdata, &t1->table[v & m1]);
            de = t1->table[v & m1];
            while (de) {
                next = de->next;
                fn(privdata, de);
                de = next;
            }

            /* Increment the reverse cursor not covered by the smaller mask.*/
            v |= ~m1;
            v = rev(v);
            v++;
            v = rev(v);

            /* Continue while bits covered by mask difference is non-zero */
        } while (v & (m0 ^ m1));
    }

    return v;
}
```



- dict涉及到字典扩容，扩容期间scan怎么处理的——判断dict的ht[0]与ht[1]哪个大，优先扫描大的那个，之后再扫描另一个
- redis-cli scan 13912 match key99* count 1000，scan后面的13912是游标的下标，表示从dict的hash[13912]处开始扫描
- 最后对遍历出来的key进行过滤，如下

```
//遍历出所有的key后，按模式进行匹配
    /* Step 3: Filter elements. */
    node = listFirst(keys);
    while (node) {
```

参考：https://xie.infoq.cn/article/c9f2e33be3dded138a58d72dc



更多的scan命令：

- scan 指令是一系列指令，除了可以遍历所有的 key 之外，还可以对指定的容器集合进行遍历。
- zscan 遍历 zset 集合元素，
- hscan 遍历 hash 字典的元素、
- sscan 遍历 set 集合的元素。



# 大Key

​	大Key的定义：一般单个String类型的Key的value大小达到10KB，集合类型的Key数超过5000



​	如何定位大Key:

- 使用scan命令扫描每一个key，使用type获取每个key的类型
- 通过相应数据结构的size/len方法获取大小，对每种类型，保留大小的Top N展示出来

​	Redis已提供相应提供可直接使用：

```
!redis-cli  --bigkeys
```



参考：http://jinguoxing.github.io/redis/2018/09/04/redis-scan/


