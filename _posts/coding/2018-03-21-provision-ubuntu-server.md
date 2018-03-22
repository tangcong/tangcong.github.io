---
layout: post
title: 配置Ubuntu Server
description: ubuntu,apt-get,systemctl,git,vim,go 
category: coding
---

## 配置软件镜像源
  不同的ubuntu版本应选择对应的镜像源,避免安装软件时出现依赖冲突等.
  推荐使用清华大学的[镜像源](https://mirror.tuna.tsinghua.edu.cn/help/ubuntu/) 

## linux平台软件包管理
  linux平台上的[软件包管理](https://www.ibm.com/developerworks/cn/linux/l-cn-rpmdpkg/)
### apt-get/deb

### yum/rpm

## linux平台启动管理

linux平台上的[启动管理](https://coolshell.cn/articles/17998.html)演化史

### sysv init

### upstart

### systemd

## linux平台日志服务

### journalctl

## 时间配置

### ntpd

### ntpstat

### tzselect

### timedatectl

### 配置文件
```
tz=Asia/Shanghai
sudo cp -vf /usr/share/zoneinfo/$tz /etc/localtime
echo $tz | sudo tee /etc/timezone
```

## Network

### VMWARE网络模式

#### Bridge

#### NAT

#### Host-Only

### Ubuntu Network Configure

#### 配置文件


```
/etc/network/interfaces
/etc/resolv.conf
```

#### auto-dhcp

#### static ip 

#### netmask/gateway

#### dns

### Network Command

#### ip addr

#### dig

#### ifdown/ifup

#### ifconfig

#### telnet

#### nslookup

#### netstat

#### nmap

### HTTP/HTTPS代理配置

#### bashrc


```
export http_proxy="xx.yy.com:port"
export https_proxy="xx.yy.com:port"
```


#### APT配置


```
/etc/apt/apt.conf.d/01proxy
Acquire::http { Proxy "http://xx.yy.com:port";}
```


## 安装常用软件(GO/GIT/VIM) 

### GO
   下载最新的stable版本[GO](https://golang.org/dl/),解压[安装](https://golang.org/doc/install?download=go1.10.linux-amd64.tar.gz)即可
   如安装了低版本的GO,先删除之

```
	sudo apt-get purge golang-go
```
### GIT

### VIM

## 配置VIM插件
 常见的VIM插件管理方法

### Manual Installation

### Vundle

### vim-plug

### pathogen

## VIM常用插件

### a.vim

### nerdtree 

### tagbar.vim

### taglist.vim

### vim-easy-align

## GO环境配置

### vim-go 

### guru

#### GoImplements
 
#### GoCallees

#### GoCallers

#### GoReferrers

### gocode

#### 将GOPATH/bin目录加入到$PATH路径,gocode基本原理是client/server模式

#### 通过&lt;C-x&gt;&lt;C-o&gt;开启自动提示


## 配置OpenSSh-Server


## 构建开发镜像

### Dockerfile
