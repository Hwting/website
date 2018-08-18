---
title: "kubernetes最佳实践S01E05：优雅地终止"
date: 2018-08-18T18:39:01+08:00
draft: false
banner: "https://cdn-images-1.medium.com/max/1600/0*CkNz3GlnZGhyycfY.jpg"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "今天是Google Developer Advocate `Sandeep Dinesh`的七部分视频和博客系列的第五部分，介绍如何充分利用您的Kubernetes环境。"
tags: ["kubernetes"]
categories: ["kubernetes"]
---

# kubernetes最佳实践S01E05：优雅地终止

* 作者：Sandeep Dinesh, Google Developer Advocate
* 日期：2018/05/18

[原文](https://cloudplatform.googleblog.com/2018/05/Kubernetes-best-practices-terminating-with-grace.html)

编者按：今天是Google Developer Advocate `Sandeep Dinesh`的七部分视频和博客系列的第五部分，介绍如何充分利用您的Kubernetes环境。

对于分布式系统，处理故障是关键。 Kubernetes通过监视系统状态并重新启动已停止执行的服务的控制器来解决这个问题。 另一方面，Kubernetes通常可以强制终止您的应用程序，作为系统正常运行的一部分。

在本期“Kubernetes最佳实践”中，让我们来看看如何帮助Kubernetes更有效地完成工作并体验下如何减少应用程序停机时间。

在容器出现之前的世界中，大多数应用程序在VM或物理机器上运行。 如果应用程序崩溃，启动替换程序需要很长时间。 如果您只有一台或两台机器来运行应用程序，那么这种恢复时间是不可接受的。

相反，在崩溃时使用进程级监视来重新启动应用程序变得很常见。 如果应用程序崩溃，进程监视可以捕获退出代码并立即重新启动应用程序。

随着像Kubernetes这样的系统的出现，不再需要进程监控系统，因为Kubernetes会重启崩溃的应用程序本身。 Kubernetes使用事件循环来确保容器和节点等资源是健康的。 这意味着您不再需要手动运行这些进程监视器。 如果资源未通过运行状况检查，Kubernetes会自动轮转更换。

[查看这一集视频，了解如何为您的服务设置自定义健康检查。](https://www.youtube.com/watch?v=mxEvAPQRwhw)

## Kubernetes终止生命周期

Kubernetes不仅可以监控应用程序的崩溃。 它可以创建更多应用程序副本，以便在多台计算机上运行，更新应用程序，甚至可以同时运行多个版本的应用程序！

这意味着Kubernetes可以终止一个完全健康的容器有很多原因。 如果您使用滚动更新更新部署，Kubernetes会在启动新pod时慢慢终止旧pod。 如果释放节点，Kubernetes将终止该节点上的所有pod。 如果节点资源不足，Kubernetes将终止pod以释放这些资源

[查看第三集，可以了解有关资源的更多信息](https://cloudplatform.googleblog.com/2018/05/Kubernetes-best-practices-Resource-requests-and-limits.html)

重要的是，您的应用程序要优雅地处理终止，以便最终用户受到的影响最小，并且恢复时间尽可能快([Time-to-recovery](https://en.wikipedia.org/wiki/Mean_time_to_recovery))！

实际上，这意味着您的应用程序需要处理SIGTERM消息并在收到它时开始关闭。 这意味着你需要保存所有需要保存的数据，关闭网络连接，完成剩下的任何工作以及其他类似任务。

一旦Kubernetes决定终止您的pod，就会发生一系列事件。 让我们看看Kubernetes终止生命周期的每一步。

### 1.Pod被设置为“终止”状态，并从所有服务的端点列表中删除

此时，pod停止获得新的流量。 在pod中运行的容器不会受到影响。

### 2. `preStop Hook`被执行

`preStop Hook`是一个特殊命令或http请求，发送到pod中的容器。

如果您的应用程序在接收SIGTERM时没有正常关闭，您可以使用此`Hook`来触发正常关闭。 接收SIGTERM时大多数程序都会正常关闭，但如果您使用的是第三方代码或管理系统则无法控制，所以`preStop Hook`是在不修改应用程序的情况下触发正常关闭的好方法。

### 3. SIGTERM信号被发送到pod

此时，Kubernetes将向pod中的容器发送SIGTERM信号。 这个信号让容器知道它们很快就会被关闭。

您的代码应该监听此事件并在此时开始干净地关闭。 这可能包括停止任何长期连接（如数据库连接或WebSocket流），保存当前状态或类似的东西。

即使您使用`preStop Hook`，如果您发送SIGTERM信号，测试一下应用程序会发生什么情况也很重要，这样您在生产环境中才不会感到惊讶！

### 4. Kubernetes优雅等待期

此时，Kubernetes等待指定的时间称为优雅终止等待期。 默认情况下，这是30秒。 值得注意的是，这与`preStop Hook`和SIGTERM信号并行发生。 Kubernetes不会等待`preStop Hook`完成。

如果你的应用程序完成关闭并在`terminationGracePeriod`完成之前退出，Kubernetes会立即进入下一步。

如果您的Pod通常需要超过30秒才能关闭，请确保增加优雅等待期。 您可以通过在Pod的YAML中设置`terminationGracePeriodSeconds`选项来实现。 例如，要将其更改为60秒：

![gcp-terminationGracePeriodSeconds.png](https://4.bp.blogspot.com/-0DCYTsMreLE/Wvxz1mWgZlI/AAAAAAAAFr8/ZiLqOMsh-rEB7Tom2bGowC6qOrxjr5qlgCEwYBhgL/s320/gcp-terminationGracePeriodSeconds.png)

### 5. SIGKILL信号被发送到pod，并删除pod

如果容器在优雅等待期结束后仍在运行，则会发送SIGKILL信号并强制删除。 此时，所有Kubernetes对象也会被清除。

## 结论

Kubernetes可以出于各种原因终止pod，并确保您的应用程序优雅地处理这些终止，这是创建稳定系统和提供出色用户体验的核心。