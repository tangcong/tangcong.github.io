---
layout: post
title: kubernetes best practice 
description: k8s,kubernetes,practice,docker,image,deployment,high availbility
category: coding
---

## 背景 

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
