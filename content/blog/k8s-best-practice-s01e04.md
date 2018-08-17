---
title: "kubernetes最佳实践S01E04：资源请求和限制"
date: 2018-08-18T01:42:12+08:00
draft: false
banner: "https://cdn-images-1.medium.com/max/1600/0*tiEgL6ffA_sqY7Ub.jpg"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "今天是Google Developer Advocate `Sandeep Dinesh`关于如何充分利用Kubernetes环境的七部分视频和博客系列的第四部分。"
tags: ["kubernetes"]
categories: ["kubernetes"]
---

# kubernetes最佳实践S01E04：资源请求和限制

* 作者：Sandeep Dinesh, Google Developer Advocate
* 日期：2018/05/11

[原文](https://cloudplatform.googleblog.com/2018/05/Kubernetes-best-practices-Resource-requests-and-limits.html)

编者按：今天是Google Developer Advocate `Sandeep Dinesh`关于如何充分利用Kubernetes环境的七部分视频和博客系列的第四部分。

当Kubernetes调度Pod时，容器是否有足够的资源来实际运行是很重要的。 如果大型应用程序被调度到资源有限的节点上，则节点可能会耗尽内存或CPU资源，并且可能会停止工作！

应用程序有可能占用比其应占有的资源更多的资源。 这可能是因为一个团队调整为更多的副本，而不是人工减少延迟（嘿，调整更多副本比让你的代码更高效容易得多！），或者一个错误的配置修改使CPU占用100％，进而导致程序失去控制。 无论问题是由糟糕的开发人员，或者糟糕的代码，亦或是运气不好引起的，重要的是你能掌控着你自己。

在本期Kubernetes最佳实践中，让我们来看看如何使用资源请求和限制来解决这些问题。

## 请求和限制

请求和限制是Kubernetes用于控制CPU和内存等资源的机制。 请求是保证容器得到的。 如果容器请求资源，Kubernetes将仅将其调度到可以为其提供该资源的节点上。 另一方面，限制确保容器永远不会超过某个值。 容器只允许达到限制资源值，然后无法获得更多资源。

重要的是要记住，限制永远不会低于请求。 如果你试试这个，Kubernetes将抛出一个错误，不会让你运行容器。

请求和限制基于单个容器。 虽然Pod通常包含一个容器，但通常也会看到Pods包含多个容器。 Pod中的每个容器都有自己的限制和请求，但由于Pod总是被认为是一个组，因此您需要将每个容器的限制和请求加在一起以获取Pod的聚合值。

要控制容器可以拥有的请求和限制，可以在Container级别和Namespace级别设置配额。 如果您想了解有关命名空间的更多信息，请参阅我们博客系列中的上一篇文章！

让我们看看这些是如何工作的。

## 容器设置

有两种类型的资源：CPU和内存。 Kubernetes调度程序使用这些来确定运行pod的位置。

[请点击这里获取这些资源的文档。](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/)

如果您在`Google Kubernetes Engine`中运行，则默认名称空间已经为您设置了一些请求和限制。

![google-namespace-requests-limits.png](https://4.bp.blogspot.com/-x4ek0a8eNio/WvS5OlYKMsI/AAAAAAAAFnU/P7kUguuL0tYLBnAsU_wXCc2BWFhu0MzowCEwYBhgL/s1600/google-namespace-requests-limits.png)

这些默认设置适用于“Hello World”，更改它们以适合您的应用非常重要。

资源的典型`Pod spec`可能看起来像这样。 这个pod有两个容器：

![gcp-container-pod-spec.png](https://1.bp.blogspot.com/-hkQeBSBm-VE/WvS5a593XJI/AAAAAAAAFnY/mNjePyLiLJYTYrHcU_D8rOVrpfiSNYTuwCEwYBhgL/s400/gcp-container-pod-spec.png)

Pod中的每个容器都可以设置自己的请求和限制，这些都是附加的。 因此在上面的示例中，Pod的总请求为500 mCPU，内存为128 MiB，总内存为1 CPU和256MiB。

### CPU

CPU资源以毫秒定义。 如果您的容器需要运行两个完整的核心，那么您将设置值“2000m”。 如果您的容器只需要1/4的核心，那么您将设置一个“250m”的值。

关于CPU请求要记住的一件事是，如果您输入的值大于最大节点的核心数，则永远不会调度您的pod。 假设您有一个需要四个核心的Pod，但您的Kubernetes群集由双核VM组成 - 您的pod将永远不会被调度！

除非您的应用程序专门用于利用多个核心（科学计算和某些数据库），否则通常最好将CPU请求保持在“1”或更低，并运行更多副本以扩展它。 这为系统提供了更大的灵活性和可靠性。

就CPU限制而言，事情变得有趣。 CPU被认为是“可压缩”资源。 如果您的应用程序开始达到您的CPU限制，Kubernetes会开始限制您的容器。 这意味着CPU将受到人为限制，使您的应用程序性能可能更差！ 但是，它不会被终止或退出。 您可以使用`liveness`探针的运行状况检查来确保性能未受影响。

### 内存

内存资源以字节为单位定义。 通常，你给内存一个[mebibyte](https://en.wikipedia.org/wiki/Mebibyte)值（这基本上与兆字节相同），实际上你可以提供从字节到PB的任何单位。

和CPU一样，如果您输入的内存请求大于节点上的内存量，则你的pod永远不会被调度。

与CPU资源不同，内存无法压缩。 因为没有办法限制内存使用量，如果容器超过其内存限制，它将被终止。 如果您的pod由Deployment，StatefulSet，DaemonSet或其他类型的控制器管理，则控制器会轮转替换。

### 节点

请务必记住，您无法设置大于节点提供的资源的请求。 例如，如果您拥有一个双核群集，具有2.5核心请求的Pod则永远不会被调度到这里！ 您可以在[此处](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-architecture#node_allocatable_resources)找到Kubernetes Engine VM相关的文档资源。

## 命名空间设置

在一个理想的世界里，Kubernetes的容器设置足以照顾好一切，但世界是一个黑暗而可怕的地方。 人们很容易忘记设置资源，或者流氓团队可以设置非常高的请求和限制，并占用超过他们公平份额的群集。

要防止出现这些情况，可以在命名空间级别设置ResourceQuotas和LimitRanges。

### ResourceQuotas

创建命名空间后，可以使用ResourceQuotas将其锁定。 ResourceQuotas非常强大，但我们只看看如何使用它们来限制CPU和内存资源的使用。

资源配额可能如下所示：

![gcp-resourcequota.png](https://4.bp.blogspot.com/-Cdi_H12oezU/WvTDkgavI8I/AAAAAAAAFoc/TK_QHCyX-eYJ8iyikprFooI4yunszy_5gCEwYBhgL/s1600/gcp-resourcequota.png)

看一下这个例子，你可以看到有四个部分。 配置每个部分都是可选的。

requests.cpu是命名空间中所有容器的最大组合CPU请求（以毫秒为单位）。 在上面的示例中，您可以拥有50个具有10m请求的容器，5个具有100m请求的容器，甚至一个具有500m请求的容器。 只要命名空间中请求的总CPU小于500m！

requests.memory是命名空间中所有容器的最大组合内存请求。 在上面的示例中，您可以拥有50个具有2MiB请求的容器，5个具有20MiB CPU请求的容器，甚至是具有100MiB请求的单个容器。 只要命名空间中请求的总内存小于100MiB！

limits.cpu是命名空间中所有容器的最大组合CPU限制。 它就像requests.cpu，但是这里指的是限制。

limits.memory是命名空间中所有容器的最大组合内存限制。 它就像requests.memory，但是同样地这里指的是极限。

如果您使用的是生产和开发命名空间（与每个团队或服务的命名空间不同），则常见的模式是在生产命名空间上没有配额，在开发命名空间上没有严格的配额。 这使得生产能够在流量激增的情况下获取所需的所有资源。

### LimitRange

您还可以在命名空间中创建LimitRange。 与名称空间作为整体查看的配额不同，LimitRange适用于单个容器。 这有助于防止人们在命名空间内创建超小容器或超大容器。

LimitRange可能如下所示：

![gcp-limit-range.png](https://4.bp.blogspot.com/-7rDxM4FRyBM/WvS6cjtCVxI/AAAAAAAAFns/xj7vbf0mWqA9OGpb86lKjzt9j4o2-3FBQCEwYBhgL/s400/gcp-limit-range.png)

看一下这个例子，你可以看到有四个部分。 同样，设置每个部分都是可选的。

`default section`设置容器中容器的默认限制。 如果在limitRange中设置这些值，则任何未明确设置这些值的容器都将被分配默认值。

`defaultRequest section`设置pod中容器的默认请求。 如果在limitRange中设置这些值，则任何未明确设置这些值的容器都将被分配默认值。

`max section`将设置Pod中容器可以设置的最大限制。 默认部分不能高于此值。 同样，在容器上设置的限制不能高于此值。 请务必注意，如果设置了此值且默认部分未设置，则任何未自行显式设置这些值的容器都将被指定为最大值作为限制。

`min section`设置Pod中容器可以设置的最小请求。 defaultRequest部分不能低于此值。 同样，在容器上设置的请求也不能低于此值。 请务必注意，如果设置了此值且defaultRequest部分未设置，则min值也将成为defaultRequest值。

## Kubernetes Pod的生命周期

Kubernetes调度程序使用这些资源请求来运行您的工作负载。 了解其工作原理非常重要，这样您才能正确调整容器。

假设您想要在群集上运行Pod。 假设Pod Spec有效，Kubernetes调度程序将使用round-robin负载平衡来选择节点来运行您的工作负载。

注意：例外情况是，如果使用nodeSelector或类似机制强制Kubernetes在特定位置安排Pod。 使用nodeSelector时仍会发生资源检查，但Kubernetes只会检查具有所需标签的节点。

然后Kubernetes检查节点是否有足够的资源来满足Pod容器上的资源请求。 如果没有，则移动到下一个节点。

如果系统中的所有节点都没有剩余资源来填充请求，那么Pod将进入“挂起”状态。 通过使用节点自动缩放器等Kubernetes Engine功能，Kubernetes Engine可以自动检测此状态并自动创建更多节点。 如果有多余的容量，`autoscaler`也可以减少和删除节点，以节省您的钱！

但限制怎么处理？ 如您所知，限制必须高于请求。 如果您有一个节点，其中所有容器限制的总和实际上高于机器上可用的资源，该怎么办？

在这一点上，Kubernetes进入了一种被称为“过度使用状态”的东西。这是事情变得有趣的地方。 由于CPU可以被压缩，因此Kubernetes将确保您的容器获得他们请求的CPU并且将限制其余部分。 内存无法压缩，因此如果Node耗尽内存，Kubernetes需要开始决定终止哪些容器。

让我们想象一下我们有一台机器内存不足的情况。 Kubernetes会做什么？

注意：Kubernetes 1.9及以上版本如下。 在以前的版本中，它使用稍微不同的过程。 请参阅[此文档](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/kubelet-eviction.md#eviction-strategy)以获得深入的解释。

Kubernetes寻找使用比他们要求的更多资源的Pod。 如果您的Pod的容器没有请求，那么默认情况下它们使用的数量超过了他们的要求，因此这些是终止的主要候选者。 其他主要候选人是已经超过他们的要求但仍然在他们的限制之下的容器。

如果Kubernetes发现多个已经超过其请求的pod，则它将按Pod的优先级对这些进行排名，并首先终止最低优先级的pod。 如果所有Pod具有相同的优先级，Kubernetes将终止最多资源请求的Pod。

在非常罕见的情况下，Kubernetes可能会被迫终止仍在其请求范围内的Pod。 当关键系统组件（如kubelet或docker）开始占用比为它们保留的资源更多的资源时，就会发生这种情况。

## 结论

虽然您的Kubernetes集群可以在不设置资源请求和限制的情况下正常工作，但随着团队和项目的增长，您将开始遇到稳定性问题。 添加对您的Pod和命名空间的请求和限制只需要一点额外的努力，并且可以避免您遇到许多令人头疼的问题！

