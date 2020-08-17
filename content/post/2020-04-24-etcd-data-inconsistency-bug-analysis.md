---
layout: post
title: 重启etcd可能导致数据不一致BUG分析
description: etcd,raft,boltdb,data inconsistency
date:     2020-04-24
author:     "唐聪"
categories: [ "Tech"]
tags:
    - etcd
---

​

## 重启etcd可能导致数据不一致BUG分析

### 背景

近期我们遇到一个严重BUG，开启鉴权后，重启etcd就可能导致数据不一致，根本原因是鉴权相关操作未做幂等性，consistent index未持久化，重启会导致命令重放，
进而导致鉴权版本号不一致，放大导致mvcc数据不一致，客户端表现写进去数据读取不到。

问题详细描述如下:
[issue](https://github.com/etcd-io/etcd/issues/11651)

为了解决以上问题以及提高后续定位不一致问题的效率，我们提了以下几个PR。

* 修复consistent index未持久化的bug.
[auth/store: save consistentIndex to fix a data corruption bug](https://github.com/etcd-io/etcd/pull/11652)

* 重构consistent index.
[refactor consistentindex](https://github.com/etcd-io/etcd/pull/11699)

* 在若干关键处增加错误日志定位数据不一致.
[optimize auth/etcdserver logs to facilitate troubleshooting data inconsistency](https://github.com/etcd-io/etcd/pull/11670)

* auth status接口增加authRevision字段以提高鉴权版本号的可观测性和可测试性.
[add auth revision to AuthStatus to improve observability and testability](https://github.com/etcd-io/etcd/pull/11659)

### 内容

详情参考详细[分析文章](https://mp.weixin.qq.com/s/VJi1jzTK2G7bH1pi4ND7Yw).
