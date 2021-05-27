---
layout: post
title: "kubernetes架构分析"
subtitle: 'kubernetes架构分析'
author: "odin"
header-style: text
tags:
  - kubernetes
---

![]({{site.baseurl}}/img/in-post/post-kubernetes/kubernetes-post-head.png)

## 1. 架构原理

Kubernetes 最初源于谷歌内部的 Borg，提供了面向应用的容器集群部署和管理系统。Kubernetes 的目标旨在消除编排物理 / 虚拟计算，网络和存储基础设施的负担，并使应用程序运营商和开发人员完全将重点放在以容器为中心的原语上进行自助运营。Kubernetes 也提供稳定、兼容的基础（平台），用于构建定制化的 workflows 和更高级的自动化任务。 Kubernetes 具备完善的集群管理能力，包括多层次的安全防护和准入机制、多租户应用支撑能力、透明的服务注册和服务发现机制、内建负载均衡器、故障发现和自我修复能力、服务滚动升级和在线扩容、可扩展的资源自动调度机制、多粒度的资源配额管理能力。 Kubernetes 还提供完善的管理工具，涵盖开发、部署测试、运维监控等各个环节。

## 2.Borg简介

Borg 是谷歌内部的大规模集群管理系统，负责对谷歌内部很多核心服务的调度和管理。Borg 的目的是让用户能够不必操心资源管理的问题，让他们专注于自己的核心业务，并且做到跨多个数据中心的资源利用率最大化。
Borg 主要由 BorgMaster、Borglet、borgcfg 和 Scheduler 组成，如下图所示

![]({{site.baseurl}}/img/in-post/post-kubernetes/borg.png)

* BorgMaster 是整个集群的大脑，负责维护整个集群的状态，并将数据持久化到 Paxos 存储中；
* Scheduer 负责任务的调度，根据应用的特点将其调度到具体的机器上去；
* Borglet 负责真正运行任务（在容器中）；
* borgcfg 是 Borg 的命令行工具，用于跟 Borg 系统交互，一般通过一个配置文件来提交任务。

## 3.Kubernetes架构

Kubernetes 借鉴了 Borg 的设计理念，比如 Pod、Service、Labels 和单 Pod 单 IP 等。Kubernetes 的整体架构跟 Borg 非常像，如下图所示

![]({{site.baseurl}}/img/in-post/post-kubernetes/architecture.png)

kubernetes主要由以下几个核心组件：

* `etcd`：保存整个集群的状态
* `kube-apiserver`:提供统一并且是唯一的操作资源api入口，具备完整的认证、授权、访问控制、注册和发现功能
* `kube-controller-manager`:负责维护集群的状态，比如故障检测、自动扩缩容、更新等功能
* `kube-scheduler`:负责整个集群的资源调度
* `kubelet`:主要负责节点容器的生命周期管理，同时也负责Volume(CVI)和网络(CNI)管理
* `Container runtime`:负责镜像管理以及pod和容器的真正运行(CRI),默认的容器是Docker
* `kuber-proxy`:负责为Service提供集群内的代理，服务发现和负载均衡

![]({{site.baseurl}}/img/in-post/post-kubernetes/components.png)

除了核心组件，还有一些推荐的Add-ons(插件):

* `kube-dns`:负责为整个集群提供DNS服务
* `ingress controller`:为服务提供外网入口
* `heapster`:提供资源监控
* `Dashboard`:提供 GUI
* `Federation`:提供跨可用区的集群
* `Fluentd-elasticsearch`:提供集群日志采集、存储与查询

## 4.分层架构

Kubernetes 设计理念和功能其实就是一个类似 Linux 的分层架构，如下图所示

![]({{site.baseurl}}/img/in-post/post-kubernetes/fenceng.png)

* `ecosystem生态层`:在接口层之上的庞大容器集群管理调度的生态系统，可以划分为两个范畴:
  * `kubernetes内部`:CRI、CNI、CVI、镜像仓库、Cloud Provider、集群自身的配置和管理等
  * `kubernetes外部`:日志、监控、配置管理、CI、CD、Workflow、FaaS、OTS 应用、ChatOps等
* `interface layer(接口层)`:kubectl 命令行工具、客户端 SDK 以及集群联邦
* `govemance layer(管理层)`:系统度量（如基础设施、容器和网络的度量），自动化（如自动扩展、动态 Provision 等）以及策略管理（RBAC、Quota、PSP、NetworkPolicy 等）
* `application layer(应用层)`:部署（无状态应用、有状态应用、批处理任务、集群应用等）和路由（服务发现、DNS 解析等）
* `nucleus(核心层)`:Kubernetes 最核心的功能，对外提供 API 构建高层的应用，对内提供插件式应用执行环境

## 核心组件

![]({{site.baseurl}}/img/in-post/post-kubernetes/core-packages.png)

## 核心API

![]({{site.baseurl}}/img/in-post/post-kubernetes/core-apis.png)

## 生态系统

![]({{site.baseurl}}/img/in-post/post-kubernetes/core-ecosystem.png)