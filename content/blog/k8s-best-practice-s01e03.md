---
title: "K8s Best Practice S01e03"
date: 2018-08-12T23:02:08+08:00
draft: false
banner: "https://cdn-images-1.medium.com/max/1600/0*rtpkUtv66zDCEUUX.jpg"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "今天是Google Developer Advocate `Sandeep Dinesh`关于如何充分利用Kubernetes环境的七部分视频和博客系列的第三部分。"
tags: ["kubernetes"]
categories: ["kubernetes"]
---

# kubernetes最佳实践S01E03：设置带readiness和liveness探针的健康检查

* 作者：Sandeep Dinesh, Google Developer Advocate
* 日期：2018/05/04

[原文](https://cloudplatform.googleblog.com/2018/05/Kubernetes-best-practices-Setting-up-health-checks-with-readiness-and-liveness-probes.html)

编者按：今天是Google Developer Advocate `Sandeep Dinesh`关于如何充分利用Kubernetes环境的七部分视频和博客系列的第三部分。

分布式系统很难管理。 一个重要原因是有许多动态部件都为系统运行起作用。 如果一个小部件损坏，系统必须检测它，绕过它并修复它。 这一切都需要自动完成！

健康检查(Health Check)是让系统知道您的应用实例是否正常工作的简单方法。 如果您的应用实例不再工作，则其他服务不应访问该应用或向其发送请求。 相反，应该将请求发送到已准备好的应用程序实例，或稍后重试。 系统还应该能够使您的应用程序恢复健康状态。

默认情况下，当pod中的所有容器启动时，Kubernetes开始向pod发送流量，并在崩溃时重新启动容器。 虽然这在开始时可以“足够好”，但您还可以通过创建自定义运行状况检查来使部署更加健壮。 幸运的是，Kubernetes使这个相对简单，所以没有理由不去这么干！

在本期Kubernetes最佳实践中，让我们了解`readiness`和`liveness`探针的细节，何时使用哪种探针，以及如何在Kubernetes集群中进行设置。

## 健康检查的类型

Kubernetes为您提供两种类型的健康检查，了解两者之间的差异及其用途非常重要。

### Readiness

`Readiness`探针旨在让Kubernetes知道您的应用何时准备好其流量服务。 Kubernetes确保`readiness`探针检测通过，然后允许服务将流量发送到pod。 如果`readiness`探针开始失败，Kubernetes将停止向该容器发送流量，直到它通过。

### Liveness

`Liveness`探针让Kubernetes知道你的应用程序是活着还是死了。 如果你的应用程序还活着，那么Kubernetes就不管它了。 如果你的应用程序已经死了，Kubernetes将删除Pod并启动一个新的替换它。

## 健康检查是如何提供帮助的？

让我们看看两个场景，`readiness`探针和`liveness`探针可以帮助您构建鲁棒性更强的应用程序。

### Readiness

让我们假设您的应用需要一分钟的时间来预热并开始。 即使该过程已启动，您的服务在启动并运行之前也无法运行。 如果要将此部署扩展为具有多个副本，也会出现问题。 新副本在完全就绪之前不应接收流量，但默认情况下，Kubernetes会在容器内的进程启动后立即开始发送流量。 通过使用`readiness`探针，Kubernetes等待应用程序完全启动，然后才允许服务将流量发送到新副本。

![google-kubernetes-probe-readiness.gif](https://1.bp.blogspot.com/-bGrr80erBA8/Wut2YBZDLfI/AAAAAAAAFk0/OS-BsPYJDPgdWYavTVk6NTnS3Y_FwjqkwCLcBGAs/s1600/google-kubernetes-probe-readiness.gif)

### Liveness

让我们假设另一种情况，你的应用程序有一个令人讨厌的死锁情况，导致它无限期挂起并停止提供请求服务。 因为该服务还在运行，默认情况下Kubernetes认为一切正常并继续向已经broken的pod发送请求。 通过使用`liveness`探针，Kubernetes会检测到应用程序不再提供请求并重新启动有问题的pod。

![google-kubernetes-probe-liveness.gif](https://4.bp.blogspot.com/-60edbfCN6Y4/Wut2h4omBHI/AAAAAAAAFk4/P_9lB5GZG8cIhrLRcdaLCp0oRVPnZHFqwCLcBGAs/s1600/google-kubernetes-probe-liveness.gif)

## 探针类型

下一步是定义测试`readiness`和`liveness`的探针。 有三种类型的探测：HTTP，Command和TCP。 您可以使用它们中的任何一个进行`liveness`和`readiness`检查。

### HTTP

HTTP探针可能是最常见的自定义`liveness`探针类型。 即使您的应用程序不是HTTP服务，您也可以在应用程序内创建轻量级HTTP服务以响应`liveness`探针。 Kubernetes去`ping`一个路径，如果它得到的是200或300范围内的HTTP响应，它会将应用程序标记为健康。 否则它被标记为不健康。

您可以在[此处](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#define-a-liveness-http-request)阅读有关HTTP探针的更多信息。

### Command

对于`Command`探针，Kubernetes则只是在容器内运行命令。 如果命令以退出代码0返回，则容器标记为健康。 否则，它被标记为不健康。 当您不能或不想运行HTTP服务时，此类型的探针则很有用，但是必须是运行可以检查您的应用程序是否健康的命令。

您可以在[此处](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#define-a-liveness-command)阅读有关Command探针的更多信息。

### TCP 

最后一种类型的探针是TCP探针，Kubernetes尝试在指定端口上建立TCP连接。 如果它可以建立连接，则容器被认为是健康的; 否则被认为是不健康的。

如果您有HTTP探针或Command探针不能正常工作的情况，TCP探测器会派上用场。 例如，gRPC或FTP服务是此类探测的主要候选者。

您可以在[此处](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#define-a-tcp-liveness-probe)阅读有关TCP探针的更多信息。

## 配置探针的初始化延迟时间

可以通过多种方式配置探针。 您可以指定它们应该运行的频率，成功和失败阈值是什么，以及等待响应的时间。 有关[配置探针的文档](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#configure-probes)非常清楚地介绍了其不同的选项及功能。

但是，使用`liveness`探针时需要配置一个非常重要的设置，就是`initialDelaySeconds`设置。

如上所述，`liveness`探针失败会导致pod重新启动。 在应用程序准备好之前，您需要确保探针不会启动。 否则，应用程序将不断重启，永远不会准备好！

我建议使用[p99延迟](https://www.quora.com/What-is-p99-latency)启动时间作为initialDelaySeconds，或者只是取平均启动时间并添加一个缓冲区。 随着您应用的启动时间变得越来越快，请确保更新这个数值。

## 结论 

大多数人会告诉你健康检查是分布式系统的基本要求，Kubernetes也不例外。 使用健康检查为您的Kubernetes服务奠定了坚实的基础，更好的可靠性和更长的正常运行时间。 值得庆幸的是，Kubernetes让您轻松做到这些！





