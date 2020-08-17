---
layout: post
title: 深入理解zookeeper(工程实现篇)
description: zookeeper架构，zookeeper 数据模型,核心概念(watches,session), get/create/set等核心接口读写流程分析，zookeepr c api 实现等
date:     2016-09-18
author:     "唐聪"
categories: [ "Tech"]
tags:
    - zookeeper
---

## zookeeper概况

### 背景&问题

在生产环境中，为了提高服务可用性、支撑更多的用户量等,分布式应用服务都会在不同IDC多个节点上部署，我们很可能会遇到以下问题:

* 分布在各个机器、IDC的应用程序如何能高效读取、修改配置?
* 当配置变更时,各节点应用如何快速发现变化、及时响应处理?
* 如何在应用程序部署的各个节点中，选举一个节点，作为leader,执行协调相关操作? leader挂掉时，其他节点能重新发起leader选举? 如何避免脑裂？ 如何处理网络分区？
* 当某个节点异常挂掉时，如何及时发现？

第1、2点在节点数较少、对性能要求不高的情况下，我们可以通过将配置存储在mysql+定时轮询解决。若对性能要求较高我们就需要结合cache、agent、配置变更notify、mysql等组件实现一套配置系统来解决，如淘宝的[diamond](http://codemacro.com/2014/10/12/diamond/)。

第3、第4点，在复杂的分布式环境中，我们会遇到高网络延时、网络波动、磁盘故障、机器宕机、机器半死不活、机房断电、网络分区等一系列问题，同时要避免数据不一致、脑裂，还要追求高吞吐、低延时，最大程度减少因选举leader导致服务不可用时间等，最糟糕的是分布式理论FLP(consensus is impossible with asynchronous systems and even one failure)、CAP（consistency,high availability,partition-tolerance)告诉我们在设计上需要权衡取舍，这些如果让业务应用程序来处理，就具有一定的复杂性。

因此，这类复杂问题不适合应用程序自己解决，应用程序需要一个God,一个值得信赖的Oracle，同时God提供的Service应该尽量简单、易理解、高性能、易扩展。

Yahoo的工程师们为了解决应用这些问题，设计实现了zookeeper，为什么叫zookeeper? 因为yahoo内部不少分布式系统命名是动物名字，同时分布式环境中的复杂、混乱跟动物园(zoo)是不是有点类似?而zookeeper就是维持、管理整个动物园的秩序,这就是zookeeper的名称的来历。

[ZooKeeper](https://zookeeper.apache.org/doc/trunk/zookeeperOver.html)是一个分布式管理服务，可为应用提供配置管理、名称服务、状态同步、集群管理等功能，我们的应用场景主要是配置管理、分布式锁。

* 为什么apache给它定义是个分布式协调式服务，而不是存储服务？它的设计目标定位是什么? 
* 它可以当存储服务来使用吗？ 它的数据模型是怎样的? 如何持久化存储的? 
* zookeeper服务端的读写流程是怎样实现的？
* zookeeper c api是如何实现的？
* 如何通过zookeeper实现分布式锁、Leader Election? 
* 在生产环境实践中我们遇到了哪些问题？如何优化zookeeper性能? 对zookeeper进行监控？

本文将结合zookeeper源码(3.4.6)、在生产环境实践经验，通过分析以上问题来深入理解zookeeper,以及分享我们在实践中遇到的问题及经验。

首先,我们一窥zookeeper全貌，了解下其总体架构及设计目标。

### zookeeper架构

![zookeeper架构](/img/zkservice.jpg)

zookeeper集群节点数量一般由奇数个节点组成，节点角色由follower和leader组成，所有写请求需转发到leader,每次写请求需集群一半以上节点应答成功才写入成功，读请求在任意一台follower节点上都可以处理，只要集群中有一半以上的节点存活、并能相互通信，zookeeper集群就可以持续提供服务，因此具有较高的可用性。节点数量越多，可用性会进一步提高，但是会影响写性能，因此在生产环境中一般部署5台。
zookeeper提供的接口类似nosql系统，常用的接口有get/set/create/getchildren等，接口简单易用。

### zookeeper设计目标

#### 简单
zookeeper数据模型简单，易懂，类似文件系统的层次树形数据结构,存储数据未做shard分散到多机,而是各单机完整存储整个树形层次空间上所有路径的节点数据，数据全部保存在内存，因此可提供高吞吐量、低延迟的服务,也意味着zookeeper不适合保存大节点数据。

#### 高可用、高性能读
因单机上保存了所有数据，若没有多机之间数据同步复制机制，zookeeper系统可用性将极低，因此zookeeper在设计上一个重要目标是可复制的，各节点通过zookeeper atomic broadcast算法选举leader,同步数据。所有写请求follower节点都需转发给leader,读请求在任意一台follower节点上都可以处理。

#### 有序
zookeeper通过基于tcp连接、写请求由leader处理等机制提供有序保证，基于有序机制，zookeeper可以提供同步原语，实现分布式锁等机制。

## zookeeper数据模型

存储系统常见的[数据模型](https://en.wikipedia.org/wiki/Database_model)有关系型表格型(Relational Model)、层次树型(Hierarchical model)、扁平型(Flat model)、网络型(Network Model)、对象型(Object-oriented Model).

![五种常见存储模型图](/img/480px-Database_models.jpg)

zookeeper的数据模型是层次型,类似文件系统，但是zookeeper的设计目标定位是简单、高可靠、高吞吐、低延迟的内存型存储系统，因此它的value不像文件系统那样会适合保存大的值，官方建议保存的value大小要小于1M，提供的接口类似nosql存储系统(key是路径)。

![zookeeper层次模型](/img/zknamespace.jpg)

那么zookeeper的层次模型是通过什么数据结构实现的呢? get、set、getchildren的时间复杂度又分别是多少呢?
通过阅读zookeeper server源码，zookeeper是基于ConcurrentHashMap实现的,path是key,value是DataNode,DataNode保存了value、children、
stat等信息。

	zookeeper database模型的调用链路
	
	ZKDatabase
	 DataTree
	  ConcurrentHashMap<String, DataNode> nodes = new ConcurrentHashMap<String, DataNode>();
	   DataNode
	    data,acl,stat,children
	
	class Stat {
		long czxid;      // created zxid
		long mzxid;      // last modified zxid
		long ctime;      // created
		long mtime;      // last modified
		int version;     // version
		int cversion;    // child version
		int aversion;    // acl version
		long ephemeralOwner; // owner id if ephemeral, 0 otw
		int dataLength;  //length of the data in the node
		int numChildren; //number of children of this node
		long pzxid;      // last modified children
	}

[ConcurrentHashMap](https://www.ibm.com/developerworks/cn/java/java-lo-concurrenthashmap/)是线程安全的hash table,采用了锁分段技术来减少锁竞争,以提高性能。其结构如下图所示，由两部分组成,Segment和HashEntry,锁的粒度是Segment,每个Segment 对象包含整个散列映射表的若干个桶,散列冲突时通过链表来解决.

![ConcurrentHashMap](/img/concurrenthashmap.jpg)

因此zookeeper在使用ConcurrentHashMap时其各接口期望时间复杂度如下:

* get:O(1) 
* create/set:O(1) 
* getchildren:O(1) 

## zookeeper持久化存储

从数据模型我们知道zookeeper所有数据都是加载都内存，基于ConcurrentHashMap构建一颗DataTree,那么zookeeper要保证机器重启数据不丢失就需要实现持久化存储，而zookeeper的持久化实现是通过snapshot、txnlog实现的，snapshot是zookeeper内存数据的完整镜像,zookeeper在运行中会定时生成,txnlog是快照时间点之后的事物日志,zookeeper在重启时,通过snapshot和txnlog重建DataTree.
下图是运行中的zookeeper集群的生成的数据文件。

![zookeeper数据文件](/img/zk_file.png)

snapshot和log文件分布保存在哪？保留多少个snapshot和log文件? 什么时候清理废弃的snapshot和log 文件？
这些都可以通过在zookeeper的zoo.cfg配置文件中指定，dataDir指定snapshot路径,dataLogDir指定事物日志路径，事物日志对zk吞吐量、延时有着非常大的延时，建议datadir与dataLogDir使用不同的设备，避免磁盘IO资源的争夺，影响整个系统性能和稳定性。autopurge.snapRetainCount项表示保留多少个snapshot,每个snapshot快照清理间隔小时可以通过autopurge.purgeInterval来指定。

snapshot的生成和log文件的写入是在SyncRequestProcessor类中实现的，事物日志类TxnLog,快照类FileSnap,事物日志会追加到TxnLog,当记录数大于1000会刷到磁盘，当写入log数大于snapCount/2+randRoll(nextInt(snapCount/2)时,会开启线程将DataTree dump到磁盘,具体实现逻辑如下:

	if (zks.getZKDatabase().append(si)) {
		logCount++;
		if (logCount > (snapCount / 2 + randRoll)) {
			randRoll = r.nextInt(snapCount/2);
			// roll the log
			zks.getZKDatabase().rollLog();
			// take a snapshot
			if (snapInProcess != null && snapInProcess.isAlive()) {
				LOG.warn("Too busy to snap, skipping");
			} else {
				snapInProcess = new Thread("Snapshot Thread") {
						public void run() {
							try {
								zks.takeSnapshot();
							} catch(Exception e) {
								LOG.warn("Unexpected exception", e);
							}
						}
					};
				snapInProcess.start();
			}
			logCount = 0;
		}

	}
	toFlush.add(si);
	if (toFlush.size() > 1000) {
		flush(toFlush);
	}
	
从zookeeper持久化的基本实现可知若写请求较大会频繁生成快照，同时因为toFlush是同步刷新数据到磁盘的,所以会影响吞吐率、延时,这也是为什么txnlog建议使用性能较好的存储硬件的原因(如SSD)。

## zookeeper核心角色及概念

### leader

### follower

### observer

### session

### watcher

### access control

## zookeeper server读写流程分析

在zookeeper的服务端实现中,通过抽象出leader、follower、observer共性特点，读写请求的处理流程可以按照功能拆分成各阶段(pipeline)，每个processor负责处理其中一个阶段，采用设计模式的职责链形式，一个processor处理完，通过队列分发到下一个processor中。processor相当于工厂各元部件，而leader、follower、observer只是使用、组装的各元部件不一致，但他们可以高度复用相同的元部件，精简实现，减少代码冗余。

### 职责链处理类介绍

#### PrepRequestProcessor

此处理类根据请求的命令(create,set等)负责生成事物请求信息数据结构request,统计正在进行的事物等。

#### FollowerRequestProcessor

此处理类负责将写请求分发给leader.

#### CommitProcessor

#### SyncRequestProcessor

如前面持久化存储所述，此处理类负责持久化存储，将批量事物日志刷新到磁盘和定时生成快照。

#### SendAckRequestProcessor

此处理类在收到写请求提议后，回复ACK给leader.

#### ProposalRequestProcessor

此处理类负责将所有写请求转发给follower节点。

#### ToBeAppliedRequestProcessor



#### FinalRequestProcessor

此处理类如名字所言，是请求流行线式处理最后一环，负责处理查询请求(从zkdatabase的DataTree读取数据)和写事务请求。

### zookeeper读流程

### zookeeper写流程

## zookeeper c api


## 总结


#### 参考资料
* [Apache Zookeeper](https://zookeeper.apache.org/doc/r3.4.6/zookeeperOver.html)
* [Apache ZooKeeper: the making of](https://developer.yahoo.com/blogs/hadoop/apache-zookeeper-making-417.html)
* [ZooKeeper学习之server端实现的基本骨架](http://damacheng009.iteye.com/blog/2085968)