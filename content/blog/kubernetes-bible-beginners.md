---
title: "初学者的kubernetes圣经"
date: 2018-11-02T11:26:40+08:00
draft: false
banner: "https://www.level-up.one/wp-content/uploads/2018/07/blogimage-kubernetes-rfc.png"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "大家好，欢迎来到关于Kubernetes的Level Up系列文章。 在开始撰写本文之前，我想问你几个问题。 您或您的团队是否需要使用Kubernetes进行容器编排？ 你想学习Kubernetes是否很困惑从哪里开始？ 你愿意改变你的组织吗？ 您想简化容器软件编排吗？ 然后我想告诉你，这篇文章是所有这些问题的答案。"
tags: ["Kubernetes"]
categories: ["Kubernetes"]
---

# 初学者的 kubernetes 圣经

[原文](https://www.level-up.one/kubernetes-bible-beginners/)

大家好，欢迎来到关于 Kubernetes 的 Level Up 系列文章。 在开始撰写本文之前，我想问你几个问题。 您或您的团队是否需要使用 Kubernetes 进行容器编排？ 你想学习 Kubernetes 是否很困惑从哪里开始？ 你愿意改变你的组织吗？ 您想简化容器软件编排吗？ 然后我想告诉你，这篇文章是所有这些问题的答案。

Kubernetes 旨在简化事情，本文旨在为您简化 Kubernetes！ Kubernetes 是一个由 Google 开发的强大的开源系统。 它是为在集群环境中管理容器化应用程序而开发的。 Kubernetes 已经获得了普及，并且正在成为在云上部署软件的新标准。

学习 Kubernetes 并不困难（如果导师很好），它提供了强大的能力。 学习曲线有点陡峭。 因此，让我们以简化的方式学习 Kubernetes。 本文介绍了 Kubernetes 的基本概念，架构，它是如何解决问题的等。

## Kubernetes 是什么？

![1_vM3OxOXOYHgkJcvfE9wALA.png](https://www.level-up.one/wp-content/uploads/2018/07/1_vM3OxOXOYHgkJcvfE9wALA.png)

实际上 Kubernetes 本身就是一个用于在多台机器上运行和协调应用程序的系统。 该系统管理容器化应用程序和服务的生命周期。 为了管理生命周期，它使用不同的方法来提高可预测性，可伸缩性和高可用性。

Kubernetes 用户可以自由决定应用程序的运行和通信方式。 还允许用户扩展/缩减服务，执行滚动更新，在不同应用程序版本之间切换流量等。 Kubernetes 还提供用于定义/管理应用程序的不同接口和平台原语。

## Kubernetes 硬件

Kubernetes 需要不同类型的硬件。 我们需要了解的是 Kubernetes 本身是不需要硬件的，但功能系统需要硬件。

### Nodes

Kubernetes 中的节点是什么？ Kubernetes 中最小的计算单位之一被称为节点。 它是一台机器，位于一个集群中。 节点不一定需要是物理机器或硬件的一部分。 它可以是物理机器或虚拟机。 对于数据中心，节点是物理机器。对于 Google Cloud Platform，节点是虚拟机。 目前，我们正在讨论 Kubernetes 的硬件， 所以让我们相应地考虑一下。 但是不要将节点限制为“硬件”。

![1-3.png](https://www.level-up.one/wp-content/uploads/2018/07/1-3.png)

任何机器中总是有一个抽象层。 但在这里，没有必要担心机器的特性。 实际上，我们可以简单地将每台机器视为一组 CPU 和 RAM 资源。 这些计算机位于群集中，可以根据需要使用其资源。 当我们谈到 Kubernetes，你自然地这样想！ 在您自由地利用任何机器资源的那一刻，系统变得无比灵活。 现在任何机器都可以成为 Kubernetes 集群中的一个节点。

### Cluster

我们已经讨论了节点，对吗？ 他们似乎是小而可爱的处理单位。 他们在自己的小房子里工作。 所有这些听起来都很完美，但它还不是 Kubernetes 的方式！ 所以集群来了。 您无需担心单个节点的状态，因为它们是集群的一部分。 例如，如果单个节点表现不佳，则应该有人来管理所有这些。 此外，集群是多个节点的集合。 将所有节点的资源集中在一起，共同构成一个强大的机器。

![module_02_first_app.png](https://www.level-up.one/wp-content/uploads/2018/07/module_02_first_app.png)

集群是很智能的。 你知道为什么吗？ 当程序部署到集群上时，它会动态处理分发。 简而言之，它将任务分配给各个节点。 在该过程之间，如果添加或删除任何节点，则集群会根据需要移动任务。 程序员不需要专注于诸如单个机器运行的代码之类的东西等等。哦，我还记得一些非常有趣的东西。 你还记得星际迷航中的“博格 Borg”吗？ 这个名字来自哪里？ Google 内部的 Kubernetes 项目的名字就是这个。

### Persistent Volumes

如上所述，程序在集群上运行并由节点提供支持。 但它们不在特定节点上运行。 程序动态地运行。 因此，需要存储信息，并且不能将其随机存储在任何文件系统中。 为什么？ 例如，程序将数据保存到文件中。 但是后来，该程序被重新定位到另一个节点。 下次程序需要该文件时，它不会在预期的位置。 位置地址已经改变。 为了解决这个问题，与每个节点相关的传统本地存储被认为是用于保存程序的临时缓存。 但是，任何本地保存的数据都不会持续存在。

![1_kF57zE9a5YCzhILHdmuRvQ.png](https://www.level-up.one/wp-content/uploads/2018/07/1_kF57zE9a5YCzhILHdmuRvQ.png)

那么谁来永久存储数据呢？ 是的，持久性卷来永久存储它。 集群管理所有节点的 CPU 和 RAM 资源。 但是，集群不负责将数据永久存储在持久卷中。 本地驱动器和云驱动器可以像持久卷一样附加到集群。 这很类似于将外部驱动器插入集群。 持久卷提供文件系统。 它可以挂载到集群，而不与任何特定节点进行关联。

这是 Kubernetes 的硬件部分。 现在让我们转到软件部分。

## Kubernetes 软件

Kubernetes 的整体概念基于软件。 所以这是 Kubernetes 的主要部分。

### Containers

在 Kubernetes 中，程序运行在 Linux 容器。 这些容器基于预编译的镜像。 镜像可以部署在 Kubernetes 上。 你知道什么是容器化吗？ 它允许您创建 Linux 执行环境。

程序及其依赖项打包在单个镜像中，并在网上共享。 因此，任何人都可以按照需求下载镜像并将其部署在基础设施上。 只需一点设置即可轻松部署。 可以在程序的帮助下创建容器。 这使得能够形成有效的 CI/CD 管道。

容器能够处理多个程序。 但建议每个容器限制一个进程，因为这有助于排除故障。 更新容器很容易，如果它很小，部署也很容易。 最好有许多小容器，而不是大容器。

### Pod

Kubernetes 有一些独特的特性，其中之一是它不直接运行容器。 它将一个或多个容器包装成一个 pod。 pod 的概念是同一 pod 中的任何容器使用相同的资源和相同的本地网络。

好处是容器可以容易地相互通信。 它们是孤立的，但随时可以通信。

Pod 可以在 Kubernetes 中复制。 例如，应用程序变得流行并且单个 pod 无法承受负载。 此时，可以根据需要配置 Kubernetes 以部署 pod 的新副本。

但是，只有在重负载期间才需要进行复制。 Pod 也可以在正常条件下复制。 这有助于统一负载平衡和抵抗故障。

![Kubernetes-Pod.jpg](https://www.level-up.one/wp-content/uploads/2018/07/Kubernetes-Pod.jpg)

Pod 能够容纳多个容器，但如果可能的话，应该限制为一个或两个容器。 原因是 Pod 作为一个单元向上和向下扩展。 Pod 内的容器也必须与 Pod 一起缩放。 在这个阶段，单个容器的需求并不重要。 另一方面，这会导致资源浪费和昂贵的账单。

为避免这些，请将 pod 限制为少数几个容器。 如果您遇到过“sidecar”这个词，那就意味着辅助容器。 所以有主进程容器，可能有一些辅助容器。

### Depoloyment

![07751442-deployment-2.png](https://www.level-up.one/wp-content/uploads/2018/07/07751442-deployment-2.png)

你已经注意到，Pod 是 Kubernetes 的基本单位。 但它们不是直接在群集上启动的。 它们由多个抽象层进行管理，这就是 Deployment 存在的原因。 主要目的是声明一次运行的副本数。

添加 Deployment 之后，它会监控 pod 的数量。 同样，如果 pod 不存在，则 Deployment 会重新创建它。

有趣的是，通过 Deployment，可以无需处理 pod。 同时，通过声明系统状态，可以自动管理所有内容。

### 通过 Ingress 向成功进军

我们已经讨论了 Kubernetes 的所有基本概念。 使用它们，您可以创建节点、集群。 创建集群后，就可以在集群上启动 pod 的部署。 但是，您如何允许外部流量到您的应用程序？ 我们还没有讨论过这个问题。

![gate-clipart-open-school-12.jpg](https://www.level-up.one/wp-content/uploads/2018/07/gate-clipart-open-school-12.jpg)

根据 Kubernetes 的概念，它提供了 Pod 和外部世界之间的隔离。 要与在 pod 中运行的服务进行通信，需要打开一个通道。 通道是沟通的媒介。 它被称为“Ingress”。

有许多方法可以向集群添加 Ingress。 最常见的是通过 Ingress Controller 或负载均衡器。 我们可以讨论这两种方法之间的区别，但目前不需要，因为它太具技术性。

现在，您应该了解 Ingress 对于尝试 Kubernetes 非常重要。 虽然你其他都做得很对，但是如果你不考虑 Ingress，你将无法到达任何地方！

## Kubernetes 如何解决问题？

在讨论了 Kubernetes 的部署部分之后，有必要了解 Kubernetes 的重要性。

### 容器编排和 Kubernetes

容器是虚拟机。 它们轻巧，可扩展且独立。 容器放在一起需要设置安全策略，限制资源利用率等。如果您的应用程序基础结构类似于下面共享的镜像，则容器编排是必要的。

它可能是在几个容器上运行的 Nginx/Apache + PHP/Python/Ruby/Node.js 应用程序，通过数据库副本通讯。 容器编排(Container orchestration)将帮助您自己管理所有内容。

![1_ffvv2BRLtnwRfZaoEPYi_g.png](https://www.level-up.one/wp-content/uploads/2018/07/1_ffvv2BRLtnwRfZaoEPYi_g.png)

请考虑您的应用程序不断增长。 例如，您继续添加更多特性/功能，并且在某个时间点，您意识到它突然变成了巨大的单体应用。

现在，管理庞大的应用程序是不可能的，因为它会占用太多 CPU 和 RAM。 所以你最终决定将应用程序分成更小的块。 他们每个人都有一个特定的任务。 现在，您的基础架构如下所示：

![1_gGm9l6xEiZQlJgIGOpFqSw.png](https://www.level-up.one/wp-content/uploads/2018/07/1_gGm9l6xEiZQlJgIGOpFqSw.png)

因此，您需要一个带有队列系统的缓存层，以获得更好的异步性能。 现在你会发现，存在服务发现，负载平衡，运行状况检查，存储管理，自动扩展等挑战。

在所有这些情况下，谁会来帮助你？ 是的，容器编排将成为你的救星！ 原因是容器编排功能非常强大，可以解决大部分挑战。

### 那有什么选择呢？

主要候选者是 Kubernetes，AWS ECS 和 Docker Swarm。 在所有这些中，Kubernetes 是最受欢迎的！ Kubernetes 提供最大的社区。 Kubernetes 解决了所描述的所有主要问题。 它是可移植的，可运行在大多数云提供商，裸机，混合环境以及所有这些的组合上。 此外，它也是可配置和模块化的。 它提供自动放置，自动重启，自动复制和容器自动修复等功能。

## 要点

最重要的是，Kubernetes 拥有一个活跃的在线社区。 这个社区的成员在网上以及在世界主要城市线下聚会。 国际会议“KubeCon”已经证明是一个巨大的成功。 Kubernetes 还有一个官方的 Slack 小组。 Google 云平台，AWS，Azure，DigitalOcean 等主要云提供商也提供其支持渠道。 你还在等什么？

学习 Kubernetes 并不像你想象的那么困难。 我推荐 Kubernetes 视频教程系列[YouTube](https://www.youtube.com/playlist?list=PLot-YkcC7wZ9xwMzkzR_EkOrPahSofe5Q)。 它提供逐步，详细的 Kubernetes 学习。 Level Up 还提供来自 Kubernetes 的 DevOps 大师视频课程的 Learn Kubernetes[Learn Kubernetes from a DevOps guru (Kubernetes + Docker) \| Level Up](https://academy.level-up.one/p/learn-kubernetes-from-a-devops-guru-kubernetes-docker/?product_id=698613&coupon_code=LEVEL_BLOG_1)。 那么让我们开始吧！
