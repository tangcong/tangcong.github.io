---
layout: post
title: etcd数据不一致bug分析汇总
description: etcd,raft,boltdb,data inconsistency
date:     2020-03-02
author:     "唐聪"
categories: [ "Tech"]
tags:
    - etcd
---

## etcd数据不一致bug案例分析

简单总结目前几个已知的etcd数据不一致案例,以及相关排查思路，后续持续更新。

### [mvcc: fix rev inconsistency](https://github.com/etcd-io/etcd/pull/6633)

确保重启后重建的版本号不能小于已经compact的revision.

#### [mvcc/backend: Fix corruption bug in defrag #11613](https://github.com/etcd-io/etcd/pull/11613)

defrag未正常结束时会生成db.tmp临时文件，如果下次defrag不会清理，并且会复用，可能会出现各种异常场景，如重启后重启索引key增多等。

### [auth/store: save consistentIndex to fix a data corruption bug #11652](https://github.com/etcd-io/etcd/pull/11652)

auth操作重放导致auth revision不一致，进而导致mvcc数据不一致。

### [a data corruption bug in revoking lease when upgrading cluster from v3.2 to v3.3/v3.4+ #11689](https://github.com/etcd-io/etcd/issues/11689)

当从3.2及以下版本升级到3.3及以上版本的时候，如果开启了鉴权，并且使用了lease,lease ttl且较短，那么较大概率导致3.3版本的lease无对法删除，因为3.3对
lease key过期操作增加了鉴权，而3.2没有，所以当leader是3.2的时候,lease过期则导致3.3无法清理，进而导致revision不一致，最终依赖revision的相关事务写操作失败。

### 总结

如果一个特性在不同版本间不兼容，并且涉及到revision变更(考虑间接和直接因素），则很可能会引发数据不一致.比如之前我们在pr里面优化了一个接口使其
幂等性，但考虑到这个在不同版本间行为不一样，会导致auth revision不一致，最后还是将其revert了。