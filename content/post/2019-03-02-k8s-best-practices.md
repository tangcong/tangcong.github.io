---
layout: post
title: kubernetes best practices 
description: k8s,kubernetes,practice,docker,image,deployment,high availbility
date:     2019-03-02
author:     "唐聪"
categories: [ "Tech"]
tags:
    - kubernetes
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

![k8s trend](/img/k8s-trend.png)


什么是k8s? k8s有何特性? 如何用好k8s?

正如上文所说k8s(kubernetes中间8个字母简写成8)是为了解决以上容器治理运维问题，用于自动化容器化应用程序的部署、扩展和管理，其核心特性如下:

* 设计思想之声明式

	用户通过一个配置文件声明服务期待的运行状态即可,如副本数,在机器死机等异常情况下k8s会尽力确保现状与预期一致,无需人工干预处理

* 服务发现、负载均衡
		
	在k8s中部署目标服务后，其他服务可通过dns/环境变量等机制访问目标服务，k8s的kube-proxy组件实现了多种负载均衡机制，无需关注后端POD扩缩容等变化

* 自动化调度

	用户声明服务的运行所需资源和限制后，k8s的kube-scheduler组件将对集群中的所有Node进行筛选过滤，选出最佳的节点列表，无需人工费劲找机器、筛选部署
		
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

知其然知其所以然，了解k8s基本架构、工作原理有助于我们灵活运用好K8S各种特性, 在使用K8S过程中遇到问题时，能够快速进行troubleshooting，解决问题。

![k8s architecture](/img/k8s-arc.png)


如上图K8S架构图所示, k8s由以下组件组成:

### Master组件

* ETCD

	K8S架构是中心化存储，ETCD保存了整个集群的状态，ETCD中数据长什么样呢？ 以ETCD2为例, example集群下分别存储了minions/Node,deployment,service,configmap等K8S常见的工作负载类型和资源. 当我们在default namespace下新建了一个服务(deployment)部署nginx时，etcd中数据又会发生什么变化呢?

	```
	ls /cls-example

	/cls-example/apiregistration.k8s.io
	/cls-example/clusterrolebindings
	/cls-example/services
	/cls-example/storageclasses
	/cls-example/daemonsets
	/cls-example/replicasets
	/cls-example/minions
	/cls-example/ranges
	/cls-example/horizontalpodautoscalers
	/cls-example/deployments
	/cls-example/pods
	/cls-example/apiextensions.k8s.io
	/cls-example/rolebindings
	/cls-example/configmaps
	/cls-example/roles
	/cls-example/serviceaccounts
	/cls-example/secrets
	/cls-example/events
	/cls-example/controllerrevisions
	/cls-example/namespaces
	/cls-example/persistentvolumeclaims
	/cls-example/persistentvolumes
	/cls-example/statefulsets
	/cls-example/clusterroles

	get /cls-example/deployments/default/nginx
	
	{
    "kind":"Deployment",
    "apiVersion":"apps/v1",
    "metadata":{
        "name":"nginx",
        "namespace":"default",
        "uid":"df828975-09b1-11e9-af67-0a587f825bc8",
        "generation":3,
        "creationTimestamp":"2018-12-27T08:31:57Z",
        "labels":{
            "qcloud-app":"nginx"
        },
        "annotations":{
            "deployment.changecourse":"RollingBack",
            "deployment.kubernetes.io/revision":"2"
        }
    },
    "spec":{
        "replicas":3,
        "selector":{},
        "template":{},
            "spec":{
                "containers":[
                    {
                        "name":"nginx",
                        "image":"nginx:v0.1",
                        "env":[
                            {
                                "name":"PATH",
                                "value":"/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
                            }
                        ],
                        "resources":{
                            "limits":{
                                "cpu":"500m",
                                "memory":"1Gi"
                            },
                            "requests":{
                                "cpu":"250m",
                                "memory":"256Mi"
                            }
                        },
						....
                    }
                ],
            }
        },
        "strategy":{...},
    },
    "status":{
        "observedGeneration":3,
        "replicas":3,
        "updatedReplicas":3,
        "readyReplicas":3,
        "availableReplicas":3,
        "conditions":[...]
    }}

	```

	从etcd nginx返回的JSON对象可知，K8S API RESOURCE OBJECT主要由元数据metadata、规范spec和状态statu组成, 元数据metadata包含deployment name,uid,namespace等核心信息,规范spec定义了用户期待的服务运行状态，即前面所说的声明式设计思想，status是服务实际运行状态.


* Kube-ApiServer 

	kube-apiserver暴露了集群的所有API，是集群的控制面板的唯一入口，提供认证、授权、权限控制等机制,其中API由WORKLOADS APIS，SERVICE APIS，CONFIG AND STORAGE APIS,METADATA APIS,CLUSTER APIS组成，其中我们最核心的WORKLOADS APIS如下:
	
	- **WORKLOADS APIS**
		- Container v1 core
		- CronJob v1beta1 batch
		- DaemonSet v1 apps
		- Deployment v1 apps
		- Job v1 batch
		- Pod v1 core
		- ReplicaSet v1 apps
		- ReplicationController v1 core
		- StatefulSet v1 apps

	k8s 将程序运行模型抽象成无状态服务deployment、有状态服务StatefulSet、计算任务Job、定时任务CronJob、各节点运行一个副本的服务daemonset,同时提供了强大的扩展能力，基于K8S CRD等扩展机制，用户可以自定义资源对象，构建新的WORKLOADS.

* kube-controller-manager

	kube-controller-manager包含一系列控制器，如下图源码所示:

	![k8s-controller-manager](/img/k8s-cm.png)
	
	部分控制器功能如下:

	- **deployment**

		负责deployment WORKLOADS的实现.

	- **statefulset**

		负责statefulset WORKLOADS的实现.

	- **Nodelifecycle**

		负责节点故障探测等生命周期管理.

	- **horizontalpodautoscaling**

		负责POD水平自动扩容.

	上文中一直提到的控制器，其又是怎样的工作的呢？下方给出了简明的伪代码工作示意图:

	```

	while( 1 ) {
		current state <- apiserver //从apiserver中获取程序当前状态
		desired state <- apiserver //从apiserver中获取程序的期望状态
		if current state != desired state { //如果当前状态不等于期望状态
			reconcile(current state) //则进行一致性操作，使当前状态与期望状态一致
		}
	}

	```

* kube-scheduler 

	负责资源调度,按照预定的调度策略、过对节点进行筛选、打分选出最佳的节点部署POD.

### Node组件

* kubelet

	部署在各个Node上的agent,调度器将资源分配到对应的NODE后，负责POD的生命周期管理等.

* kube-proxy

	部署在各个Node上的proxy,提供服务发现和负载均衡能力。

* container runtime

	container runtime负责镜像管理以及Pod和容器的真正运行(CRI),如常用的dockerd.

## 托管型 vs 独立型

目前各大云商都提供了托管型和独立型两种集群供选择,其各自特点如下:

* 托管型Kubernetes

只需创建Node节点，Master节点由容器服务创建并托管。具备简单、低成本(master节点无需付费)、高可用、无需运维管理Kubernetes集群Master节点的特点。
托管型适合希望运维省心、追求低成本、专注业务应用的用户。

* 独立型Kubernetes

需要创建3个Master（高可用）节点及若干Node节点，可对集群基础设施进行更细粒度的控制，需要自行规划、维护、升级服务器集群。
独立型适合对k8s比较懂、master节点需定制的、有运维技术能力、成本不敏感的用户使用。

## 集群网络 

谈到K8S集群网络，首先得了解下POD的网络设计模型:

* 每个POD都有独立的IP

* Node节点上的POD无需NAT就可以与所有节点上的POD进行通信

* NODE上的Agent等组件可以与本机上的所有POD进行

* 主机网络的POD无需NAT就可以与所有节点上的POD进行通信

如何实现K8S POD的网络模型呢? TKE目前支持kubenet和cni网络插件。

### kubenet 基础网络

![k8s-kubenet-network](/img/kubenet.png)

如图所示，tke目前默认网络插件是kubenet, 当node加入集群时,controller-manager分配给每个node一个cidr, ipadm插件是host-local,由它来管理本机的ip分配及回收.

使用veth设备来连接不同namespace网络,同node不同POD间访问基于cbr0网桥,pod创建时调用kubenet网络插件,kubenet调用loopback、bridge、host-local cni标准插件，实现lo设备添加、veth设备连接到cbr0网桥、ip地址分配等.


```
brctl show cbr0

bridge name	bridge id		STP enabled	interfaces
cbr0		8000.0a58ac140301	no		veth3eb8df72
							veth438fda9b
							vethf9fe59ba

```

正如上面所描述,kubnet并不能实现跨node pod间访问，tke是如何实现跨node访问的呢?

kubenet需要基于cloud provider的vpc路由规则才能实现跨node访问，tke使用的vpc global route.
controller-manager分配cidr给加入集群的node时,会调用vpc的接口下发全局路由，比如node 1,pod cidr 172.20.1.0/24, 用户vpc任意node访问此cidr时请求都会被转发到node 1,10.0.0.10.

pod1(172.20.1.2)访问pod3(172.20.2.2)简要流程如下:

pod 1及node 1路由分别如下:


```
default via 172.20.1.2 dev eth0
172.20.1.0/24 dev eth0 scope link  src 172.20.1.2

```

```

default via 10.0.0.10 dev eth0
10.0.0.0/20 dev eth0  proto kernel  scope link  src 10.0.0.10
172.20.1.0/24 dev cbr0  proto kernel  scope link  src 172.20.1.0

```

* pod 1未匹配到172.20.2.2路由，包从默认eth0设备发出
* node 1上也未匹配到目的ip的路由规则，包从node 1默认eth0设备发出
* 母机发现此目的ip 匹配到路由172.20.2.0/24,将包发完node 2(10.0.0.11)
* 包进入node 2后匹配到cbr0的路由规则，将包分发给cbr0
* cbr0根据arp表将包转发给对应的veth设备，转发到pod 3


#### FAQ

* 加入node失败

	当pod cidr耗尽，分配失败时, 会导致无法加入新节点

* 创建service失败,无法分配ip

	tke创建集群时，会让用户填写最大pod数/service数,容器子网等，整个容器子网资源由pod cidr  + service cidr组成,若service cidr较小而service比较多时会导致创建service失败.

* 容器网段与用户vpc cidr冲突,导致用户访问异常

	创建集群时若选择的cidr与用户vpc已有cidr冲突，将导致用户vpc相关网段访问异常，容器下发的路由具有最高优先级。

#### 最佳实践

由以上问题可知，创建集群时根据需要选择合理的容器子网非常重要，尽量提前做好容量规划，避免容器与vpc cidr冲突，减少不必要的麻烦。

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

### 选择合适的Node类型 

## CI/CD 

## Security

## 日志

## 事件

## Trouble Shooting  
