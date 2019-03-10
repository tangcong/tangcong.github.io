---
layout: post
title: kubernetes best practices 
description: k8s,kubernetes,practice,docker,image,deployment,high availbility
category: coding
---

## 背景 

2013年docker横空出世，正如其口号build,ship,run,其创新地提出将程序运行环境依赖打包为镜像,解决程序部署环境依赖、可移植问题，构建镜像仓库, 实现程序快速分发交付，全球可部署，并结合linux kernel cgroup/namespace等技术，解决程序运行资源隔离、相比虚拟机更高的资源利用率等问题. 凭着以上技术优势，docker在技术圈掀起了容器化热潮，成为现今最火热的技术话题, 带动整个容器社区蓬勃发展。

虽然docker在应用程序打包、分发、快速启动并可实现轻量级资源隔离方面表现出色，但它仅仅是一个镜像构建、容器启动的工具，无法解决以下问题:

* 容器如何扩缩容？
* 容器故障如何自愈？
* 容器如何做负载均衡和服务发现？
* 容器间如何访问？
* 如何基于容器进行发布? 支持多种发布策略?
* ......

技术社区为了解决以上容器治理、运维等问题，先后诞生了k8s、docker swarm等容器平台，其中k8s是建立在google运行生产工作量15年的经验基础上(borg)，并结合社区中的最佳创意和实践而实现,swarm是docker公司推出的docker多机集群版。如下图google指数所示，docker依然是技术圈热点, 用户在k8s、swarm的选择上，google主导的k8s很明显已经成为主流.

![k8s trend](/images/myblog/k8s-trend.png)

什么是k8s? k8s有何特性? 如何用好k8s?

正如上文所说k8s(kubernetes中间8个字母简写成8)是为了解决以上容器治理运维问题，用于自动化容器化应用程序的部署、扩展和管理，其核心特性如下:

* 设计思想之声明式

	用户通过一个配置文件声明服务期待的运行状态即可,如副本数,在机器死机等异常情况下k8s会尽力确保现状与预期一致,无需人工干预处理

* 服务发现、负载均衡
		
	在k8s中部署目标服务后，其他服务可通过dns/环境变量等机制访问目标服务，k8s的kube-proxy组件实现了多种负载均衡机制，无需关注后端POD扩缩容等变化

* 自动化调度

	用户声明服务的运行所需资源和限制后，k8s的kube-scheduler组件将对集群中的所有node进行筛选过滤，选出最佳的节点列表，无需人工费劲找机器、筛选部署
		
* 高度自愈能力

	POD异常时会自动重启.
	在配置健康检查后,健康检查失败后会kill容器重启.
	无状态服务当机器死机等异常情况下会自动调度到新节点.

* 自动化扩容 

	当服务cpu等资源达到用户配置的阈值后将触发自动化扩容，低于预期时进行缩容操作，告别人工扩容时代
	集群节点资源不顾时，也可触发集群NODE扩容，避免因资源不够而导致的部署、扩容失败等

* 自动化发布和回滚

	k8s将常用的无状态逻辑服务抽象成一种Workload,命名为Deployment, 基于其实现了自动化的增量更新等多种策略的发布机制,并且在发布过程支持暂停、恢复。
	
* 秘钥和配置管理
	
	k8s通过secret和configmap来管理程序所需要的敏感信息和常用的配置文件。

* 存储编排

	Kubernetes的PVC(Persistent Volume Claim)对象实现了将不同云、不同分布式存储系统，提供的存储能力抽象成统一的接口,实现了资源提供方和使用方解耦。

K8S特性丰富强大的同时也意味着其实现复杂度相比swarm要高很多，用户对其把控难度也加大，我们在使用其强大功能、提高生产力的同时，又该如何避免采坑、灵活用好K8S呢？

本文将从k8s原理初探、集群选型(托管型、集群型)、集群网络、K8S工作负载类型、服务发现、负载均衡、运行时、调度器、监控、日志、存储、成本优化等方面进行简要总结.

## K8S原理

## 托管型 vs 独立型

### 托管型

### 独立型

## 集群网络 

### kubenet 基础网络

### cni 高级网络

### 网络规划

### 网络策略

## 应用

### POD

### Deployment

### Statefulset

### JOB/CRONJOB

### CRD

### TIPS

#### IMAGE

镜像核心技术分层、写时复制(COW)

* docker images
根据镜像命查出镜像ID
```
docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             latest              d8233ab899d4        3 weeks ago         1.2MB

/var/lib/docker//aufs/imagedb/content/sha256/d8233ab899d419c58cf3634c0df54ff5d8acc28f8173f09c21df4a07229e1205

根据镜像ID查出每层ID
```

docker inspect d8233ab899d4

```

"RootFS": {
		"Type": "layers",
				"Layers": [
						"sha256:adab5d09ba79ecf30d3a5af58394b23a447eda7ffffe16c500ddc5ccb4c0222f"
				]
}

从layerdb里面查出真实的id

/var/lib/docker/image/aufs/layerdb/sha256/adab5d09ba79ecf30d3a5af58394b23a447eda7ffffe16c500ddc5ccb4c0222f
diff:adab5d09ba79ecf30d3a5af58394b23a447eda7ffffe16c500ddc5ccb4c0222f

cacheid:5ea6da2b6b8b51e6d328e9c141641e8840469860e6225ac8f94918728b3fcb86

从diff里面查出真实的层内容
root@VM-0-17-ubuntu:/var/lib/docker/aufs/diff# ls | xargs ls
5ea6da2b6b8b51e6d328e9c141641e8840469860e6225ac8f94918728b3fcb86:
bin  dev  etc  home  root  tmp	usr  var

运行挂载数据卷容器
docker run -it --rm --name test -v /data  busybox

root@VM-0-17-ubuntu:/var/lib/docker# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
8ca5daf4a0d7        busybox             "sh"                21 minutes ago      Up 21 minutes                           test

/var/lib/docker/containers/8ca5daf4a0d71c1f640410098b5ddf6a04a37282eb95e0e17ad6bcdbfc305099/config.v2.json

"MountPoints": {
		"/data": {
				"Destination": "/data",
						"Driver": "local",
						"ID": "c1ef4db96eac11e9c6416bf21aa46b9088b30be82e345f1e25bfcd0623851a66",
						"Name": "afeead734780b7b545b1fbc9b2d78bc0b94d792cecce031c6fedbe2ea8798767",
						"RW": true,
						"Source": "",
						"Spec": {},
						"Type": "volume"
		}
}


/ # cd /data
/data # ls
/data # touch hello
/data # touch world
/etc # echo "hello,etcd" > etcd
/etc # ls
etcd         group        hostname     hosts        localtime    mtab         network      passwd       resolv.conf  shadow
/etc # touch test

cat /var/lib/docker/image/aufs/layerdb/mounts/8ca5daf4a0d71c1f640410098b5ddf6a04a37282eb95e0e17ad6bcdbfc305099/init-id
6dd5150e7562beb04675a16b72ae38f8fce68e8a98f773f3fdfbb5d81efd2a40-init

cat /var/lib/docker/image/aufs/layerdb/mounts/8ca5daf4a0d71c1f640410098b5ddf6a04a37282eb95e0e17ad6bcdbfc305099/mount-id
6dd5150e7562beb04675a16b72ae38f8fce68e8a98f773f3fdfbb5d81efd2a40

root@VM-0-17-ubuntu:/var/lib/docker/aufs/diff# tree 6dd5150e7562beb04675a16b72ae38f8fce68e8a98f773f3fdfbb5d81efd2a40-init
6dd5150e7562beb04675a16b72ae38f8fce68e8a98f773f3fdfbb5d81efd2a40-init
├── dev
│   ├── console
│   ├── pts
│   └── shm
├── etc
│   ├── hostname
│   ├── hosts
│   ├── mtab -> /proc/mounts
│   └── resolv.conf
├── proc
└── sys

6 directories, 5 files
root@VM-0-17-ubuntu:/var/lib/docker/aufs/diff# tree 6dd5150e7562beb04675a16b72ae38f8fce68e8a98f773f3fdfbb5d81efd2a40
6dd5150e7562beb04675a16b72ae38f8fce68e8a98f773f3fdfbb5d81efd2a40
├── data
├── etc
│   ├── etcd
│   └── test
└── root

3 directories, 2 files


root@VM-0-17-ubuntu:/var/lib/docker/volumes/afeead734780b7b545b1fbc9b2d78bc0b94d792cecce031c6fedbe2ea8798767# tree
.
└── _data
    ├── hello
    └── world

1 directory, 2 files

docker commit  8ca5daf4a0d7 busybox:v1

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             v1                  2251e265458e        6 minutes ago       1.2MB
busybox             latest              d8233ab899d4        3 weeks ago         1.2MB

docker inspect 2251e265458e

"RootFS": {
		"Type": "layers",
				"Layers": [
			    "sha256:adab5d09ba79ecf30d3a5af58394b23a447eda7ffffe16c500ddc5ccb4c0222f",
				"sha256:fc0725d04c508f404ff18c87a8e97cec567071625149098814bdedbac3e1d18e"
		]
}

image/aufs/layerdb/sha256/18c355f51195ccddefab80eb9fdbd0ecc09184faf6f6b287cbbabc04319a4f35/diff:1:sha256:fc0725d04c508f404ff18c87a8e97cec567071625149098814bdedbac3e1d18e

root@VM-0-17-ubuntu:/var/lib/docker/image/aufs/layerdb/sha256/18c355f51195ccddefab80eb9fdbd0ecc09184faf6f6b287cbbabc04319a4f35# tree
.
├── cache-id
├── diff
├── parent
├── size
└── tar-split.json.gz

cat /var/lib/docker/aufs/diff/8d2106cbb0020c11873c14eb3018f33be8aa300e73cdde801d3f7ed5444dac54

/var/lib/docker/aufs/diff/8d2106cbb0020c11873c14eb3018f33be8aa300e73cdde801d3f7ed5444dac54
├── data
├── etc
│   ├── etcd
│   └── test
└── root

root@VM-0-17-ubuntu:/var/lib/docker/image/aufs/layerdb/sha256/18c355f51195ccddefab80eb9fdbd0ecc09184faf6f6b287cbbabc04319a4f35# cat parent
sha256:adab5d09ba79ecf30d3a5af58394b23a447eda7ffffe16c500ddc5ccb4c0222f

root@VM-0-17-ubuntu:/var/lib/docker/image/aufs/layerdb/sha256/18c355f1195ccddefab80eb9fdbd0ecc09184faf6f6b287cbbabc04319a4f35# cat cache-id
8d2106cbb0020c11873c14eb3018f33be8aa300e73cdde801d3f7ed5444dac54

root@VM-0-17-ubuntu:/var/lib/docker/image/aufs/layerdb/sha256/18c355f51195ccddefab80eb9fdbd0ecc09184faf6f6b287cbbabc04319a4f35# cat diff
sha256:fc0725d04c508f404ff18c87a8e97cec567071625149098814bdedbac3e1d18e


```


#### 精简镜像大小

#### 镜像版本打tag,勿使用latest

#### 配置Readiness and Liveness Probes

#### 使用Helm Charts部署

#### 不同环境、业务通过namespace隔离 

#### 灵活使用LABEL

#### 不要用root权限运行容器 

#### 日志输出到stdout,stderr 

## 服务负载均衡

### iptables

### ipvs

### 四层lb

### 七层lb

## 调度

### Node Selector

### Node Taint

### Extender Scheduler

## 存储

### CBS云盘

### NFS

### HOST PATH

### EMPTY DIR

### LOCAL PV

## 监控

### 平台告警策略

#### 容器

#### 节点

#### 服务

#### 集群

#### POD

### Promethues + Grafana

##  成本优化

### Cluster Autoscaler

### Horizontal Pod Autoscaler

### 选择合适的node类型 

## CI/CD 

## Security

## 日志

## 事件

## Trouble Shooting  
