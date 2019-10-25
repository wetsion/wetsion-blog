---
title: Redis学习：过期策略
description: Redis过期策略
categories:
 - 后端
 - Redis
tags:
 - Redis
 - 中间件
---




redis的过期策略有定期删除（主动）和惰性删除（被动）。

**（主动）定期删除**，就是redis每隔段时间就会随机抽取一些设置了过期时间的key，检查是否过期，若过期则删除，默认是每隔100毫秒。

> 这里redis具体的操作是：1.从设置了过期时间的key中随机抽取20个key做测试，2.从这20个key中找到已过期的并删除，3.如果这20个key中已过期的超过了25%则从第一步再重新开始。以上操作redis每秒执行10次。

**（被动）惰性删除**，就是在获取某个key时，redis检查一下是否设置了过期时间以及是否过期，如果过期了就会删除，并且不会返回任何东西。

但如果定时删除漏掉了一些过期key，并且后续也没查这些key，那么也就不会走惰性删除，那么这些key也就堆积在内存中了，会消耗内存，如果已使用的内存大于maxmemory，这时redis还有一个内存淘汰机制。

内存淘汰的过程是，客户端要执行新命令，例如新增，redis先检查内存使用情况，如果大于maxmemory，根据已配置的策略清除key，然后再执行新命令。

redis的内存淘汰机制提供了以下几种：

- noeviction

“不驱逐”，顾名思义，就是不会清除内存，当内存不足以写入新数据时，新写入数据会报错。

- allkeys-lru

LRU还是挺常见的，Least Recently Used，即最近最少使用，当内存不足以写入新数据时，在**所有的key**里，删除最近最少使用的key。

- allkeys-random

当内存不足以写入新数据时，从**所有的key**中，随机删除某个key

- volatile-lru

从**已设置过期时间的key**中，删除最近最少使用的key

- volatile-random

从**已设置过期时间的key**中，随机删除某个key

- volatile-ttl

从**已设置过期时间的key**中，优先删除最早要过期的key


通过命令：

> CONFIG GET maxmemory-policy

可以查看当前淘汰策略，我本地redis是4.0.10版本，默认是noeviction。

通过

> CONFIG SET maxmemory-policy allkeys-lru

可设置淘汰策略

对于maxmemory，通过`CONFIG GET maxmemory`命令查看到默认为0（*64位系统默认为0，32位系统默认为3GB内存限制*），即代表对内存使用没有限制，直到耗尽所有可用内存（当到达极限时，redis会返回内存不足的错误，但不会因为内存不足而使服务器死机），所以最好做一些限制，可以使用`CONFIG SET maxmemory xxx`设置maxmemory为指定大小。


> 另外，值得一提的是，在删除一部分key之后，redis并不是将内存直接释放回操作系统，RSS可能仍然是之前的大小，但redis的内存分配器可以重用空闲的内存块，在删除一部分key之后再写入新的key，分配器会尝试重用之前释放的内存，RSS可能会保持稳定或者不会增长很多。


---

### 扩展：Redis中LRU策略

关于redis的LRU最近最少使用策略，其实有些疑惑是如何找到最近最少使用的key，在通过查阅文档和一部分源码后有了一些了解。

```c
// server.h 里定义的 db 结构
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
} redisDb;
```

redis的LRU其实也还是随机采样的，只不过对于`allkeys-lru`和`volatile-lru`，采样的基础数据集不一样，`allkeys-lru`的基础是整个db下的所有数据（源码中的`redisDb`下的`*dict`），而`volatile-lru`的基础数据是已设置过期时间的数据（源码中的`redisDb`下的`*expires`）,然后都是从基础数据集中随机抽取`maxmemory-samples`个数据，

> `maxmemory-samples`是配置的采样个数，默认为5，可通过`CONFIG SET maxmemory-samples 10`来设置，设置的越大，采样的精度越准确，但同样也越消耗CPU，效率越低。

然后淘汰掉最少使用的key。而在`maxmemory-samples`个采样数据中，最少使用的数据该如何确定呢？

```c
// server.h 里定义的 redisObject 结构

#define LRU_BITS 24
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

可以看到，在redis中，所有的redis对象结构都有一个lru字段，是24位的unsigned类型字段，该字段的高16位用来记录访问时间，低8位用来记录访问频率，有了这个字段，就可以判断谁是最少使用的了。

---

关于redis的过期策略相关先学习到这里，还是要再结合源码多看看。
