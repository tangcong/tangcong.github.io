---
layout: post
title: 升级集群导致ETCD数据不一致分析
description: etcd,raft,boltdb,data inconsistency
date:     2020-04-19
author:     "唐聪"
categories: [ "Tech"]
tags:
    - etcd
---

​

## 升级集群导致ETCD数据不一致/毁坏案例分析

### 背景

近期我们在测试环境升级ETCD集群（3.2升级到3.3)的时候，遇到了一些奇怪现象。当集群升级完后，收到了测试同学反馈集群出现各种BUG了：

- 变更镜像不生效的
- pod不调度的
- 服务RS变更Service异常的
- 扩容不生效的
- 其他各种诡异的现象等

我们最开始怀疑K8S的controller-manager/scheduler/apiserver出bug了，一番调查后发现又是etcd出现严重问题了(恰好测试环境我们屏蔽了这个集群的告警，导致我们事后才知道),我们通过监控视图和日志观察到有如下异常：

- 部分节点db size异常增长,各个节点大小不一样
- 各节点revision不一致
- 部分etcd节点compact异常，无compact相关日志
- 各节点key数量不一致

我们尝试看日志是否为什么部分Key在有些节点有，有些节点无, 但是因为etcd的日志实在少的可怜，缺少关键的错误日志，我们一无所获，陷入迷茫和困惑中. 同时在社区上也未能搜索到有效信息，同时我们通过比对db的鉴权版本号也排除了不是我们上一次发现的严重的BUG（[https://github.com/etcd-io/etcd/issues/11651,](https://github.com/etcd-io/etcd/issues/11651)[https://github.com/etcd-io/etcd/pull/11652](https://github.com/etcd-io/etcd/pull/11652))，于是只能修改etcd代码并尝试复现了。

### 复现

首先我们采用尝试人工简单升级ETCD集群试试是否能复现，然并卵，未能复现，其实正如我们所预期一样，如果那么能复现社区早发现早解决了，不可能我们现在才遇到。

如何才能成功复现一个严重的BUG? 首先我们有如下推测：

- 跟版本升级可能有一定关联
- 跟重启可能有一定关联（上一个严重BUG是重启触发）
- 跟鉴权的版本号auth revision没任何关系(已通过比对各个节点db源数据排除)
- 观测现网监控异常时间段有哪些接口调用
- 可能需要一定的QPS触发

我们采取以下方法及技术手段对推测进行了分析和落地：

- 人工无法简单复现，我们采用混沌工程(chaos monkey)的方法来寻求自动化压测复现
- monkey压测版本是从3.2升级到3.3，每次升级一个节点，不停写入数据，然后重启，压测较长时间后再继续升级其他节点，因为升级后不可逆，不能回退
- 比对3.2和3.3版本发现两个大的改动，一个是lessor,一个是txn, 因此我们分别对3.2和3.3版本，在所有lessor和txn操作的关键路径增加debug日志和异常情况下增加详细的错误日志
- chaos monkey增加kill和restart各个节点的机制，通过简单锁机制，保证每次只杀掉1个节点，确保当前集群永远可用
- chaos monkey去除鉴权相关操作
- 根据现网监控，chaos monkey 增加txn/range/lessor/txn的各接口调用，尤其是lessor/txn QPS适当加大,lesase ttl过期时间随机生成
- 根据测试环境的监控显示我们的QPS当时在3000左右，连接数10000左右，因此我们chaos monkey也增加对应的QPS和client来满足条件
- 调度部分测试集群到压测集群，以最逼真模拟现网, 毕竟这是在k8s集群下使用下产生的

当我们把chaos monkey程序按照以上分析结论优化完后，在两套集群上同时部署，令人振奋的是，没过多久，我们就收到了一致性差异告警，我们又成功复现了。

### 原理分析

收到告警后，我们立刻采取了以下措施:

- 停止chaos moneky
- 停掉集群，避免产生大量日志干扰我们排查
- 观测数据不一致详情，发现3.2的两个节点数据一致，与3.3数据不一致
- 脚本快速分析出3.2与3.3差异的KEY
- 在各个节点中查询这个KEY的日志
- 经过搜寻，我们得到了以下关键日志

node A(3.2+,Leader)

```

637802:Mar 11 21:52:23 localhost etcd[28065]: LeaseRevoke:{LeaseRevoke 24 0  ID:2229124347589447546 } {entry-index 17 25  <nil>} {entry-term 17 2  <nil>} {rev 11 2014102077  <nil>}

637803:Mar 11 21:52:23 localhost etcd[28065]: revoke lessor,lessor id:2229124347589447546

637804:Mar 11 21:52:23 localhost etcd[28065]: revoke lessor,lessor id:2229124347589447546,key:/masterleases/A

637805:Mar 11 21:52:23 localhost etcd[28065]: apply request:header:<ID:16363671783383239993 > lease\_revoke:<ID:2229124347589447546 > ,response:&{0xc000d545e0 <nil> <nil>}

```

node B(3.2+,Follower)

```

637802:Mar 11 21:52:23 localhost etcd[28065]: LeaseRevoke:{LeaseRevoke 24 0  ID:2229124347589447546 } {entry-index 17 25  <nil>} {entry-term 17 2  <nil>} {rev 11 2014102077  <nil>}

637803:Mar 11 21:52:23 localhost etcd[28065]: revoke lessor,lessor id:2229124347589447546

637804:Mar 11 21:52:23 localhost etcd[28065]: revoke lessor,lessor id:2229124347589447546,key:/masterleases/A

637805:Mar 11 21:52:23 localhost etcd[28065]: apply request:header:<ID:16363671783383239993 > lease\_revoke:<ID:2229124347589447546 > ,response:&{0xc000d545e0 <nil> <nil>}

```

node C(3.3+,Follower)

```

1392010:Mar 11 21:56:42 localhost etcd[26312]: applyEntryNormal:check index, entry-index: 25, consistent-index: 24

1392011:Mar 11 21:56:42 localhost etcd[26312]: shouldApplyV3, entry-index: 25,  consistent-index: 25

1392012:Mar 11 21:56:42 localhost etcd[26312]: LeaseRevoke: ID:2229124347589447546 , entry-index: 25, entry-term: 2, rev: 2014102070

1392014:Mar 11 21:56:42 localhost etcd[26312]: request "header:<ID:16363671783383239993 > lease\_revoke:<id:1eef70c8a23e4f7a>" with result "error:auth: user name is empty" took too long (1.464µs) to execute

```

从日志中我们可以看到节点C(3.3+)执行lease\_revoke命令时失败了,错误是用户名为空。 此错误将继续放大，从而导致mvcc版本差异很大，进而可能无法执行txn命令，并且数据损坏。

### BUG如何产生的

为什么3.3节点执行会报user name is empty呢? 我突然想起之前比对3.2和3.3差异的时候，偶然看到了一处与其相关改动点，当时没意识到有什么问题，我立刻再去对比差异，发现正是这个PR（[https://github.com/etcd-io/etcd/pull/8031](https://github.com/etcd-io/etcd/pull/8031))在3.3的变动，导致3.3节点执行失败，在3.3以前lease revoke不需要鉴权，但是3.3这个PR增加了鉴权，导致lease过期的时候，3.3节点无法删除lease对应的key,进而出现数据不一致，版本差异，数据毁坏等一系列BUG。

详情参考issue: [https://github.com/etcd-io/etcd/issues/11689](https://github.com/etcd-io/etcd/issues/11689)

### 修复方案

我们讨论的修复方案有两种：

- 3.3以上版本，在当集群中出现不一致的版本时，兼容这种低版本的没鉴权，跳过lease revoke的鉴权流程
- 3.2版本确保lease revoke的时候auth info 不为空，用户想升级集群的时候先升级到3.2包含这个修复方案的最新版本，再继续升级到3.3版本，我们给社区提的PR也是这个方案
- 详情参考PR： [https://github.com/etcd-io/etcd/pull/11691](https://github.com/etcd-io/etcd/pull/11691)

### 最佳实践

在成功发现和修复两个严重数据不一致BUG后，我们推荐以下ETCD最佳实践方案：

- ETCD定期备份数据，推荐是至少是10分钟级别，可以使用开源的etcd-operator中的backup-operator
- 任何重要变更前一定备份数据
- 为ETCD添加自动化的一致性巡检告警，如revision差异监控、key数量差异监控
- 多关注社区issue/pr,及时升级到最新稳定版本
- 使用etcd 最新版本的3.4.4用户，推荐开启data corruption检测功能，当集群出现不一致时，拒绝集群写入，可读
- 确保集群中各节点ETCD版本一致

 最后关于ETCD数据不一致定位老大难的问题，我们也提交了PR，[https://github.com/etcd-io/etcd/pull/11670](https://github.com/etcd-io/etcd/pull/11670)，相信以后定位不一致问题会方便很多，目前已经合入master。