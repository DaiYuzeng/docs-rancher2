---
title: 工作原理
description: 本文讲述了升级 RKE 集群时，RKE 内部发生的事项，用户输入升级 RKE 集群的命令`rke up`以后，etcd 节点、controlplane 节点、worker 节点和插件在升级的过程中经历了哪些步骤。以 v1.1.0 为分界，RKE v1.1.0 及以上的版本和 RKE v1.1.0 以下的版本在升级集群的过程中触发的事项不同，请阅读对应的章节获取不同版本升级流程。
keywords:
  - rancher
  - rancher中文
  - rancher中文文档
  - rancher官网
  - rancher文档
  - Rancher
  - rancher 中文
  - rancher 中文文档
  - rancher cn
  - RKE
  - 升级指南
  - 工作原理
---

## 简介

本文讲述了升级 RKE 集群时，RKE 内部发生的事项，用户输入升级 RKE 集群的命令`rke up`以后，etcd 节点、controlplane 节点、worker 节点和插件在升级的过程中经历了哪些步骤。以 v1.1.0 为分界，RKE v1.1.0 及以上的版本和 RKE v1.1.0 以下的版本在升级集群的过程中触发的事项不同，请阅读对应的章节获取不同版本升级流程。

## RKE v1.1.0 及以上的升级流程

### 概述

RKE v1.1.0 及以上的版本提供了以下新功能：

- 支持不宕机升级，编辑或升级集群时，不会影响集群内的应用。

- 支持手动指定单个节点，并单独升级这个节点。

- 支持使用包含老版本的 Kubernetes 集群快照，将集群使用的 Kubernetes 恢复到先前的版本。这个功能使升级集群的过程变得更加安全。如果某次升级的过程中，集群中有些节点无法完成更新，导致该节点内的 pods 和应用无法访问，您可以将已经完成升级的集群所使用的 Kubernetes 降级为升级前使用的版本。

使用默认配置选项，输入`rke up`命令更新集群时，会依次触发以下事项：

1. 逐个更新每个节点的 etcd plane。

1. 逐个更新 controlplane 节点，包括 controlplane 组件和 controlplane 节点的 worker plane 组件。

1. 逐个更新每个 etcd 节点的 worker plane 组件。

1. 批量更新 worker 节点，您可以配置每批更新的节点数量。在批量更新 worker 节点的过程中，可能会有部分节点不可用，为了降低这部分节点对于升级的影响，您可以配置最大不可用节点数量。默认的最大不可用节点数量是总节点数量的 10%，最小值为 1，如果是小数则向下取整至最近的一个整数，详情请参考下文。

1. 更新[每个插件](/docs/rke/config-options/add-ons/_index)。

下文讲述运行`rke up`命令以后，每一种类的节点和插件是如何运作的。这些信息可以帮助您更好地理解和使用升级策略，在升级出现问题时，也能够帮助您进行问题排查。

### etcd 节点升级流程

升级集群的第一步是逐个升级 etcd 节点。

RKE 集群内的任何一个 etcd 节点在升级的过程中失效，会导致集群升级失败。集群的升级状态会停滞在"升级中"，无法继续执行其他 etcd 节点、controlplane 节点、worker 节点和插件的升级。

如果您的集群升级状态一直停留在“升级中”，很有可能是因为在升级 etcd 节点的过程中出现了节点失效的事件，针对这种情况进行问题排查时，请首先检查该集群中是否有 etcd 节点已经失效（node failures）。

### controlplane 节点升级流程

RKE 支持逐个升级和批量升级 controlplane 节点。RKE controlplane 节点的默认升级策略是逐个升级。您需要配置最大不可用节点数`max_unavailable_controlplane`，才可以使用批量升级 controlplane 节点的功能。只要失效的 controlplane 节点数量或百分比没有达到最大不可用节点的限制，Rancher 会继续升级 controlplane 节点，然后升级 worker 节点。

`max_unavailable_controlplane`的取值范围可以是百分比，也可以是正整数。系统会使用该数字判断每一次升级是否成功。使用百分时，表示的数量是节点总数\*百分比，然后向下取整；如果向下取整后的结果是 0~1 之间的任意小数，则会取 1 作为最后的结果。使用正整数时，请注意该数字不能大于或等于节点总数，否则在极端情况下，例如一个批次的节点全部失效，且`max_unavailable_controlplane`大于或等于节点总数，即使所有节点都升级失败，仍然会被判定为升级成功。这种取值没有意义，不仅没有简化升级流程，还给后续的问题排查添加了难度。

举个例子，如果您有 9 个 controlplane 节点，`max_unavailable_controlplane`设置为 25%，则每一次升级的节点数量是 2，因为 9 \* 25% = 2.22，向下取整得 2 。如果您有 3 个 controlplane 节点，`max_unavailable_controlplane`设置为 1%，则每次升级的节点数量是 1，因为系统要求每次升级的节点数量不小于 1。

配置批量升级和`max_unavailable_controlplane`的目的是，先完成一部分 controlplane 节点的升级，再针对剩余有问题的 controlplane 节点进行排查，然后重复升级和问题排查的过程，直至完成所有 controlplane 节点升级。相比逐个升级节点而言，批量升级效率更高，`max_unavailable_controlplane`提升了升级的容错率，不会因为单个节点失效，排查不出问题，而导致后面的所有节点都无法升级。

如果任意一个 controlplane 节点无法升级，升级流程就会停滞在这个阶段，无法进行 worker 节点升级。

### Worker 节点的升级流程

RKE 默认升级 worker 节点的方式是批量升级。每次批量升级的节点数量由`cluster.yml`文件中的最大不可用节点`max_unavailable_worker`决定。

最大不可用节点`max_unavailable_worker`可以是百分比，也可以是正整数，默认值是 10%，代表的意思是节点总数的 10%，遇到小数时向下取整。系统会使用该数字判断每一次升级是否成功。使用百分时，表示的数量是节点总数\*百分比，然后向下取整；如果向下取整后的结果是 0~1 之间的任意小数，则会取 1 作为最后的结果。

举个例子，如果您有 11 个 worker 节点，`max_unavailable_worker`设置为 25%，每一次升级的节点数量是 2，因为 11 \* 25% = 2.75，向下取整得 2 。如果您有 3 个 worker 节点，`max_unavailable_worker`设置为 1%，每次升级的节点数量是 1，因为系统要求每次升级的节点数量不小于 1。

当同一批升级的所有节点的状态转为`Ready`时，会开始下一批节点升级。如果`kubelet`和`kube-proxy`已经启动了，就代表节点的状态已经变为`Ready`。

RKE 会在开始升级之前扫描集群，寻找关机或无法连接的宿主机。如果不可用的节点超出`max_unavailable_worker`设置的值，会停止升级。

RKE 会在开始升级之前将节点标记为不可用，完成升级后将节点重新标记为可用。您也可以将 RKE 配置为升级前驱逐 pods。

RKE 首先升级节点，然后升级插件。只要不可用的节点没有超出限制，RKE 在完成节点升级后会开始升级插件。例如，如果一个集群有两个 worker 节点，升级的时候其中一个节点不可用，而最大不可用节点的数量大于 1，则不会导致升级失败，完成 1 个节点的升级后，RKE 会开始升级插件。

### 插件的升级流程

您在 pods 中部署的应用是否可用，取决于 RKE 插件是否可用。RKE 插件用于部署几个集群组件，包括网络插件，Ingress Controller，DNS 提供商，和 metrics server。

因为 RKE 插件对于允许集群接受流量来说是至关重要的，您需要批量升级插件以保持可用性。您需要在`cluster.yml`文件中为每个插件配置最大不可用副本数量以保证在升级的过程中有足够可用副本保持集群可用。

配置插件的例子，请查看[RKE 插件配置选项](/docs/rke/upgrades/configuring-strategy/_index)。

## RKE v1.1.0 之前的升级流程

### 概述

使用默认配置选项，输入`rke up`命令更新集群时，会依次触发以下事项：

1. 逐个升级 etcd 节点。

1. 批量升级 controlplane 节点。每一次升级的节点数量为 50 个或 worker 节点的总数，取两者的最小值。

1. 批量升级 worker 节点和插件，每一次升级的节点数量为 50 个或 worker 节点的总数，取两者的最小值。

1. 逐个升级插件。

### controlplane 节点和 etcd 节点的升级流程

controlplane 节点和 etcd 节点的升级策略是逐个升级。如果节点在升级的过程中不可用，会导致升级失败。

### worker 节点的升级流程

批量升级的数量 50 个或 worker 节点的总数，取两者的最小值。如果升级过程中有 worker 节点失效，会导致升级失败。

升级 worker 节点时，会重启包括`kubelet`和`kube-proxy`在内的几个 Docker 进程。当`kube-proxy`重启时，它会刷掉`iptables`。这种情况会导致无法访问该节点上的所有 pods，出现服务宕机的现象。
