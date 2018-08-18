---
title: "kubernetes最佳实践S01E07：零停机更新集群"
date: 2018-08-19T00:57:31+08:00
draft: false
banner: "https://cdn-images-1.medium.com/max/1600/0*UdMc07JcvGnbtudO.jpg"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "今天是Google Developer Advocate Sandeep Dinesh关于如何充分利用Kubernetes环境的七部分视频和博客系列的最后一部分。"
tags: ["kubernetes"]
categories: ["kubernetes"]
---

# kubernetes最佳实践S01E07：零停机更新集群

* 作者：Sandeep Dinesh, Google Developer Advocate
* 日期：2018/06/01

[原文](https://cloudplatform.googleblog.com/2018/06/Kubernetes-best-practices-upgrading-your-clusters-with-zero-downtime.html)

编者注：今天是Google Developer Advocate Sandeep Dinesh关于如何充分利用Kubernetes环境的七部分视频和博客系列的最后一部分。

每个人都知道，保持应用程序最新以优化安全性和性能是一种很好的做法。 Kubernetes和Docker可以更轻松地执行这些更新，因为您可以使用更新构建新容器并相对轻松地部署它。

就像您的应用程序一样，Kubernetes不断获得新功能和安全更新，因此底层节点和Kubernetes基础架构也需要保持最新。

在本期Kubernetes最佳实践中，让我们来看看Google Kubernetes Engine如何让您的Kubernetes集群轻松升级！

## 集群的两个部分:Master和Node

在升级群集时，需要更新两个部分：主服务器和节点。 需要首先更新主服务器，然后节点随后。 让我们看看如何使用Kubernetes Engine升级它们。

### 零停机更新Masters

Kubernetes Engine会在发布点发布时自动升级主服务器，但通常不会自动升级到新版本（例如，1.7到1.8）。 准备好升级到新版本后，只需单击Kubernetes Engine控制台中的升级主按钮即可。

![gcp-kubernetes-engine-auto-upgrade.gif](https://2.bp.blogspot.com/-U4-DcLmfv1k/WxCZqadlYHI/AAAAAAAAFyU/_SWRu2xCUBAte8OOGXKsSqJ1fqQ4QCitgCLcBGAs/s1600/gcp-kubernetes-engine-auto-upgrade.gif)

但是，您可能已经注意到该对话框显示以下内容：

“更改主版本可能会导致几分钟的控制平面停机。 在此期间，您将无法编辑此群集。”

当主服务器关闭以进行升级时，`deployments`，`services`将继续按预期工作。 但是，任何需要Kubernetes API的东西都会停止工作。 这意味着kubectl停止工作，使用Kubernetes API获取有关群集停止工作的信息的应用程序，基本上您无法在群集升级时对群集进行任何更改。

那么如何更新主机而不会导致停机？

#### 具有Kubernetes Engine区域集群的高可用Masters

虽然标准的“区域”Kubernetes Engine集群只有一个主节点支持它们，但您可以创建“区域”集群，提供多区域，高可用性的主服务器。

注意：[Kubernetes Engine区域集群最近普遍可用](https://www.google.com/url?q=https://cloudplatform.googleblog.com/2018/06/Regional-clusters-in-Google-Kubernetes-Engine-are-now-generally-available.html&sa=D&ust=1529539428270000&usg=AFQjCNEbYmCJC70VWRhq3bTXvDZLjPhJEQ)

创建群集时，请务必选择“区域”选项：

![gcp-kubernetes-engine-regional-cluster.png](https://3.bp.blogspot.com/-_hooEEGQ94k/WxCZ5TJC-ZI/AAAAAAAAFyY/RCN1Fes4z10DAQ3EL1S02OmTY23pLaJ6ACLcBGAs/s1600/gcp-kubernetes-engine-regional-cluster.png)

就是这样！ Kubernetes引擎自动在三个区域中创建节点和主控，主控器位于负载平衡的IP地址后面，因此Kubernetes API将在升级期间继续工作。

### 零停机更新Nodes

升级节点时，您可以使用几种不同的策略。 我想关注两个：

1. 滚动更新
2. 使用节点池迁移

#### 滚动更新

更新Kubernetes节点的最简单方法是使用滚动更新。 这是Kubernetes Engine用于更新节点的默认升级机制。

滚动更新以下列方式工作。 一个接一个，一个释放，一个缓冲，直到该节点上不再运行pod。 然后删除该节点，并使用更新的Kubernetes版本创建新节点。 该节点启动并运行后，将更新下一个节点。 这一直持续到所有节点都更新为止。

您可以通过在节点池上启用自动节点升级，让Kubernetes Engine完全为您管理此过程。

![gcp-kubernetes-engine-auto-node.png](https://3.bp.blogspot.com/-KcCwZAPSPs4/WxCaNNSAjJI/AAAAAAAAFyo/ad9zoG_vSfgOPs0QbMS1jNe2lFI8Dm1ewCLcBGAs/s1600/gcp-kubernetes-engine-auto-node.png)

如果您不选择此选项，Kubernetes Engine仪表板会在升级可用时提醒您：

![gcp-kubernetes-engine-dashboard-alert.png](https://1.bp.blogspot.com/-l1Rqu18_ALo/WxCaDc8KBcI/AAAAAAAAFyg/222PmcOIFg0a3Rs5XzWBuvAMPRDyUCaogCLcBGAs/s1600/gcp-kubernetes-engine-dashboard-alert.png)

只需单击该链接，然后按照提示开始滚动更新。

警告：确保您的pod由ReplicaSet，Deployment，StatefulSet或类似的东西管理。 独立Pod不会被重新调度！

虽然在Kubernetes Engine上执行滚动更新很简单，但它有一些缺点。

一个缺点是您在群集中获得的容量节点少一个。 通过扩展节点池以添加额外容量，然后在升级完成后将其缩小，可以轻松解决此问题。

滚动更新的完全自动化特性使其易于操作，但您对该过程的控制较少。 如果出现问题，还需要时间回滚到旧版本，因为您必须停止滚动更新然后撤消它。

#### 使用节点池迁移

您可以创建新节点池，等待所有节点运行，然后一次在一个节点上迁移工作负载，而不是像滚动更新那样升级“活跃”节点池。

我们假设我们的Kubernetes集群现在有三个VMs。 您可以使用以下命令查看节点：

```shell
kubectl get nodes
NAME                                        STATUS  AGE
gke-cluster-1-default-pool-7d6b79ce-0s6z    Ready   3h
gke-cluster-1-default-pool-7d6b79ce-9kkm    Ready   3h
gke-cluster-1-default-pool-7d6b79ce-j6ch    Ready   3h
```

##### 创建新的节点池

要创建名为`pool-two`的新节点池，请运行以下命令：

```shell
gcloud container node-pools create pool-two
```

注意：请记住此自定义命令，以便新节点池与旧池相同。 如果需要，还可以使用GUI创建新节点池。

现在，如果您检查节点，您会注意到有三个节点具有新池名称：

```shell
$ kubectl get nodes
NAME                                        STATUS  AGE
gke-cluster-1-pool-two-9ca78aa9–5gmk        Ready   1m
gke-cluster-1-pool-two-9ca78aa9–5w6w        Ready   1m
gke-cluster-1-pool-two-9ca78aa9-v88c        Ready   1m
gke-cluster-1-default-pool-7d6b79ce-0s6z    Ready   3h
gke-cluster-1-default-pool-7d6b79ce-9kkm    Ready   3h
gke-cluster-1-default-pool-7d6b79ce-j6ch    Ready   3h
```

但是，pod仍然在旧节点上！ 让我们来迁移pod到新节点上。

##### 释放旧节点池

现在我们需要将工作负载迁移到新节点池。 让我们以滚动的方式一次迁移一个节点。

首先，`cordon`(隔离)每个旧节点。 这将阻止新的pod安排到它们上面。

```shell
kubectl cordon <node_name>
```

一旦所有旧节点都被封锁，就只能将pod调度到新节点上。 这意味着您可以开始从旧节点中删除pod，Kubernetes会自动在新节点上调度它们。

警告：确保您的pod由ReplicaSet，Deployment，StatefulSet或类似的东西管理。 独立pod不会被重新调度！

运行以下命令以释放每个节点。 这将删除该节点上的所有pod。

```shell
kubectl drain <node_name> --force
```

释放节点后，确保新的pod已启动并运行，然后再转到下一个节点。

如果您在迁移过程中遇到任何问题，请取消旧池的保护，然后封锁并释放新池。 pod会被重新调度回旧池。

##### 删除旧节点池

一旦所有pod安全地重新调度，就可以删除旧池了。

将`default-pool`替换为要删除的池。

```shell
gcloud container node-pools delete default-pool
```

您刚刚成功更新了所有节点！


## 结论

通过使用Kubernetes Engine，您只需点击几下即可使Kubernetes群集保持最新状态。

如果您没有使用像Kubernetes这样的托管服务，您仍然可以将滚动更新或节点池方法与用在您自己的群集升级上。 不同之处在于您需要手动将新节点添加到群集中，并自行执行主升级，这可能很棘手。

我强烈建议使用Kubernetes Engine区域集群来实现高可用Master和自动节点升级，以获得无烦恼的升级体验。 如果您需要对节点更新进行额外控制，则使用节点池可以为您提供该控制，而不会放弃Kubernetes Engine为您提供的托管Kubernetes平台的优势。

到这里，我们要结束关于Kubernetes最佳实践的系列文章的第一季了。 如果您对希望我解决的其他主题有所了解，可以在[Twitter](https://twitter.com/sandeepdinesh)上找到我。








