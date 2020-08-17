---
layout: post
title: redis/codis troubleshooting总结 
description: redis,codis,aof,replication,rdb,client,troubleshooting 
date:     2018-04-02
author:     "唐聪"
categories: [ "Tech"]
tags:
    - redis
---

## redis主备同步失败 

线上某两实例重新建立主备关系时,出现如下错误
```
23203:S 02 Apr 14:27:53.148 * MASTER <-> SLAVE sync started
23203:S 02 Apr 14:27:53.148 * Non blocking connect for SYNC fired the event.
23203:S 02 Apr 14:27:53.148 * Master replied to PING, replication can continue...
23203:S 02 Apr 14:27:53.148 * Partial resynchronization not possible (no cached master)
23203:S 02 Apr 14:27:53.514 * Full resync from master: 6f4bf5d4394cdbb3fe9a2b7d9d5b0243595c2975:1189450456093
23203:S 02 Apr 14:31:15.072 * MASTER <-> SLAVE sync: receiving 6312956914 bytes from master
23203:S 02 Apr 14:31:36.156 # I/O error trying to sync with MASTER: connection lost
23203:S 02 Apr 14:31:36.786 * Connecting to MASTER x.y.z.z:6379
23203:S 02 Apr 14:31:36.786 * MASTER <-> SLAVE sync started
```

一般出现主备同步失败原因是以下参数设置不当.

### repl-backlog-size

Set the replication backlog size. The backlog is a buffer that accumulates
slave data when slaves are disconnected for some time, so that when a slave
wants to reconnect again, often a full resync is not needed, but a partial
resync is enough, just passing the portion of data the slave missed while
disconnected.

The bigger the replication backlog, the longer the time the slave can be
disconnected and later be able to perform a partial resynchronization.
The backlog is only allocated once there is at least a slave connected.

```
repl-backlog-size 1mb
```
默认较小，可根据实际情况适当增大.


### client-output-buffer-limit

The client output buffer limits can be used to force disconnection of clients
that are not reading data from the server fast enough for some reason (a
common reason is that a Pub/Sub client can't consume messages as fast as the
publisher can produce them).

The limit can be set differently for the three different classes of clients:

```
normal -> normal clients including MONITOR clients
slave  -> slave clients
pubsub -> clients subscribed to at least one pubsub channel or pattern
```

The syntax of every client-output-buffer-limit directive is the following:

```
client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>
```

A client is immediately disconnected once the hard limit is reached, or if
the soft limit is reached and remains reached for the specified number of
the limit for 10 seconds.

Both the hard or the soft limit can be disabled by setting them to zero.

```
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
```
默认较小，可根据实际情况适当增大.

[ISSUE](https://github.com/CodisLabs/codis/issues/185)

## redis-port迁移数据失败

[ISSUE](https://github.com/CodisLabs/codis/issues/318)


## redis内存碎片


## codis dashboard无法踢掉故障proxy

## codis proxy故障处理

## codis dashboard故障处理

## redis故障处理

## redis性能异常排查

### slowlog

hgetall等慢查询操作导致业务毛刺等.

### bigkeys

大key读写访问导致毛刺

### disk io

主备同步或数据备份时磁盘IO上升,影响业务.

### swap

业务反馈凌晨期间成功率较低.

## codis数据迁移

## codis数据倾斜

### 大key导致数据倾斜

### crc32算法分布不均

## redis主备数据不一致

redis重新建立主备关系后，发现主备KEY数量不一致.

![redis主备key数量不一致](/img/redis_inconsistency.jpg)

redis使用的lru算法是lazy随机式淘汰，而主备同步后，加载RDB文件时，会检查过期时间，过期时间小于当前时间会清除.

```
        /* Check if the key already expired. This function is used when loading
         * an RDB file from disk, either at startup, or when an RDB was
         * received from the master. In the latter case, the master is
         * responsible for key expiry. If we would expire keys here, the
         * snapshot taken by the master may not be reflected on the slave. */
        if (server.masterhost == NULL && expiretime != -1 && expiretime < now) {
            decrRefCount(key);
            decrRefCount(val);
        } else {
            /* Add the new object in the hash table */
            dbAdd(db,key,val);

            /* Set the expire time if needed */
            if (expiretime != -1) setExpire(NULL,db,key,expiretime);
            if (lfu_freq != -1) {
                val->lru = (LFUGetTimeInMinutes()<<8) | lfu_freq;
            } else {
                /* LRU idle time loaded from RDB is in seconds. Scale
                 * according to the LRU clock resolution this Redis
                 * instance was compiled with (normaly 1000 ms, so the
```
