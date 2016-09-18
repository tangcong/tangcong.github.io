---
layout: post
title: ZooKeeper概要及实践
description: zookeeper架构，zookeeper 数据模型,核心概念(watches,session), get/create/set等核心接口读写流程分析，zookeepr c api 使用及Trouble Shooting,最佳实践经验等
category: coding
---

## zookeeper概要


## zookeeper数据模型


## zookeeper核心概念


## zookeeper server读写流程分析


## zookeeper c api


## zookeeper c api trouble shooting

### zookeeper c api无法重连

我们团队主要开发语言是c++,因此使用的是zookeeper c api,并且基于c api封装了一个简单、易用的c++库，之前线上某业务服务使用一个老zookeeper c++库时，遇到一个诡异的BUG，业务服务运行了一段时间后,有一定的概率出现一直读取配置失败。通过lsof、netstat发现根本原因是，业务读取配置异常时，业务服务与zookeeper cluster的连接丢失了且无法自动重连上zookeeper cluster，只有手动重启服务才能恢复。我们使用的这个zookeeper c++ 库已经两三年了? 为什么这个BUG到现在才发现呢? 对比此业务服务和其他服务，此业务服务使用场景不一样，是根据请求实时读取配置,跟我们之前其他服务的用法（只有重启时从zookeeper上读取一次配置)有所不同,所以这BUG到现在才发现。

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
随后猜测是不是有其他状态漏处理了? 这个可通过出现异常的时候zhandle的状态是什么就可以判断是否有漏处理或者出现了其他非法状态，遗憾的是这c++库在核心关键的函数里面没有打印相关日志（哈哈，日志的重要性,不言而喻).
于是我们增加了日志，并且加强了处理逻辑，对于未知的状态也close zhandle,重建连接，发到线上后，bug重现后，通过日志确认的确是出现了未知的状态值0，从上面zookeeper c api定义的状态可知，0是没有定义的，通过搜索源代码发现了给zhandle state赋值为0的地方，在google也搜索到了相关[issue](https://issues.apache.org/jira/browse/ZOOKEEPER-2519).




## zookeeper 实践总结