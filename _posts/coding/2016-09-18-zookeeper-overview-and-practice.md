---
layout: post
title: ZooKeeper概要及实践
description: zookeeper架构，zookeeper 数据模型,核心概念(watches,session), get/create/set等核心接口读写流程分析，zookeepr c api 使用及Trouble Shooting,最佳实践经验等
category: coding
---

## zookeeper概要


## zookeeper数据模型

存储系统常见的[数据模型](https://en.wikipedia.org/wiki/Database_model)有关系型表格型(Relational Model)、层次树型(Hierarchical model)、扁平型(Flat model)、网络型(Network Model)、对象型(Object-oriented Model).
![五种常见存储模型图](/images/myblog/480px-Database_models.jpg]

zookeeper的数据模型是层次型,类似文件系统，但是zookeeper的设计目标定位是简单、高可靠、高吞吐、低延迟的内存型存储系统，因此它的value不像文件系统那样会适合保存大的值，官方建议保存的value大小要小于1M，提供的接口类似nosql存储系统(key是路径)。
！[zookeeper层次模型](/images/myblog/zknamespace.jpg)

那么zookeeper的层次模型是通过什么数据结构实现的呢? get、set、getchildren的时间复杂度又分别是多少呢?
通过阅读zookeeper server源码，zookeeper是基于ConcurrentHashMap实现的,path是key,value是DataNode,DataNode保存了value、children、
stat等信息。

	ZKDatabase
	-- DataTree
	---- ConcurrentHashMap<String, DataNode> nodes = new ConcurrentHashMap<String, DataNode>();
	------ DataNode
	-------- data,acl,stat,children
	
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
	
## zookeeper核心概念


## zookeeper server读写流程分析


## zookeeper c api


## zookeeper troubleshooting

### zookeeper c api无法重连

#### 问题背景

我们团队主要开发语言是c++,因此使用的是zookeeper c api,并且基于c api封装了一个简单、易用的c++库，之前线上某业务服务使用一个老zookeeper c++库时，遇到一个诡异的BUG，业务服务运行了一段时间后,有一定的概率出现一直读取配置失败。

#### 问题定位及分析

通过lsof、netstat发现根本原因是，业务读取配置异常时，业务服务与zookeeper cluster的连接丢失了且无法自动重连上zookeeper cluster，只有手动重启服务才能恢复。我们使用的这个zookeeper c++ 库已经两三年了? 为什么这个BUG到现在才发现呢? 对比此业务服务和其他服务，此业务服务使用场景不一样，是根据请求实时读取配置,跟我们之前其他服务的用法（只有重启时从zookeeper上读取一次配置)有所不同,所以这BUG到现在才发现。

随后排查zookeeper c++库相关代码，未发现明显处理不妥的之处，每个请求都会通过zoo_state检查zhandle状态。
首先看看zookeeper zhandle 有哪些状态(基于zookeeper 3.4.6版本)。

	/* zookeeper state constants */
	#define EXPIRED_SESSION_STATE_DEF -112
	#define AUTH_FAILED_STATE_DEF -113
	#define CONNECTING_STATE_DEF 1
	#define ASSOCIATING_STATE_DEF 2
	#define CONNECTED_STATE_DEF 3
	#define NOTCONNECTED_STATE_DEF 999
	
	const int ZOO_EXPIRED_SESSION_STATE = EXPIRED_SESSION_STATE_DEF;
	const int ZOO_AUTH_FAILED_STATE = AUTH_FAILED_STATE_DEF;
	const int ZOO_CONNECTING_STATE = CONNECTING_STATE_DEF;
	const int ZOO_ASSOCIATING_STATE = ASSOCIATING_STATE_DEF;
	const int ZOO_CONNECTED_STATE = CONNECTED_STATE_DEF;

对于无效的状态如（ZOO_EXPIRED_SESSION_STATE）都有close zhandle句柄，重新初始化,但是线上竟然还有无法重连的问题，比较诡异。
随后猜测是不是有其他状态漏处理了? 这个可通过出现异常的时候zhandle的状态是什么就可以判断是否有漏处理或者出现了其他非法状态，遗憾的是这c++库在核心关键的函数里面没有打印相关日志（日志的重要性不言而喻).

#### 问题解决

于是我们增加了日志，并且加强了处理逻辑，对于未知的状态也close zhandle,重建连接，发到线上后，bug重现后，通过日志确认的确是出现了未知
的状态值0，同时因为我们修复版本对未知状态也进行了处理，所以线上问题也得以修复。
从上面zookeeper c api定义的状态可知，0是没有定义的，通过搜索zookeeper c api源代码确认了给zhandle state赋值为0的代码处，通过google也搜索到了相关[ISSUE](https://issues.apache.org/jira/browse/ZOOKEEPER-2519).

### zookeeper cluster一节点cpu高负载


## zookeeper 实践总结