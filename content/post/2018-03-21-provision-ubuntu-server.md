---
layout: post
title: 基于Ubuntu配置高效的开发环境
description: ubuntu,apt-get,systemctl,git,vim,go 
date:     2018-03-21
author:     "唐聪"
categories: [ "Tech"]
tags:
    - ubuntu
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

在.bashrc文件里面增加以下配置.

```
export http_proxy="xx.yy.com:port"
export https_proxy="xx.yy.com:port"
```


#### APT配置

在/etc/apt/apt.conf.d路径增加以下文件及内容.
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

## MarkDown环境配置

### [vim-markdown](https://github.com/plasticboy/vim-markdown)
Syntax highlighting, matching rules and mappings for the original Markdown and extensions.

    Plugin 'godlygeek/tabular'
    Plugin 'plasticboy/vim-markdown'

### [vim-markdown-preview](https://github.com/JamshedVesuna/vim-markdown-preview)

A small Vim plugin for previewing markdown files in a browser.

    Plugin 'JamshedVesuna/vim-markdown-preview'.


### [grip](https://github.com/joeyespo/grip)

Render local readme files before sending off to GitHub.
```
    pip install grip
    grip test.md 0.0.0.0
```

### [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/)

GitHub.com uses its own version of the Markdown syntax that provides an additional set of useful features, many of which make it easier to work with content on GitHub.com.

## C/C++环境配置

### [Exuberant Ctags](http://ctags.sourceforge.net/)

Exuberant Ctags is the latest incarnation of a [family of computer programs] ctags that scan source code files to create an index of identifiers (tags) and where they are defined. Vim uses this index (a so-called tags file) to enable you to jump to the definition of any identifier using the [Control-]] ctrl_mapping mapping.

### [vim-easytags](https://github.com/xolox/vim-easytags)

Automated tag file generation and syntax highlighting of tags in Vim.

### [YouCompleteMe](https://github.com/Valloric/YouCompleteMe)

YouCompleteMe is a fast, as-you-type, fuzzy-search code completion engine for Vim.
a Clang-based engine that provides native semantic code completion for C/C++/Objective-C/Objective-C++ (from now on referred to as "the C-family languages").

### [clang_complete](https://github.com/Rip-Rip/clang_complete)

Vim plugin that use clang for completing C/C++ code.
You don't need any ctags for it to work! Only clang is needed. Clang version 2.8 or higher is recommended for c++ completion. After a . , -> and :: it is automatically trying to complete the code. :smile:

### [OmniCppComplete](https://github.com/vim-scripts/OmniCppComplete)

C/C++ omni-completion with ctags database. :broken_heart:

### [llvm](https://llvm.org/)

The LLVM Project is a collection of modular and reusable compiler and toolchain technologies. 

*   LLVM Core.
*   Clang.
*   LLDB.
*   libc++.
*   compiler-rt.
*   lld.

The LLVM Core libraries provide a modern source- and target-independent optimizer, along with code generation support for many popular CPUs (as well as some less common ones!) These libraries are built around a well specified code representation known as the LLVM intermediate representation ("LLVM IR"). The LLVM Core libraries are well documented, and it is particularly easy to invent your own language (or port an existing compiler) to use LLVM as an optimizer and code generator.

### [clang](https://clang.llvm.org/)

Clang is an "LLVM native" C/C++/Objective-C compiler, which aims to deliver amazingly fast compiles (e.g. about 3x faster than GCC when compiling Objective-C code in a debug configuration), extremely useful error and warning messages and to provide a platform for building great source level tools. The Clang Static Analyzer is a tool that automatically finds bugs in your code, and is a great example of the sort of tool that can be built using the Clang frontend as a library to parse C/C++ code.

*   Fast compiles and low memory use
*   Expressive diagnostics
*   GCC compatibility

## 构建开发镜像

### Dockerfile
