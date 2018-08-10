---
title: "Kubernetes最佳实践：第一季概览"
date: 2018-08-10T14:19:42+08:00
draft: false
banner: "https://venturebeat.com/wp-content/uploads/2018/06/kubernetes-logo-1-e1512659947545.png"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "考虑到这一点，我开始根据我在日常与人交谈时收到的问题和反馈，开展题为“Kubernetes最佳实践”（以下是幻灯片和视频）的演讲。这个演讲非常受欢迎，我决定深入研究各个主题。我最初为这次演讲制作了七集内容（我想七集是非常合适的数量），我真的认为这些文章和视频可以帮助你和你的团队迅速提升Kubernetes。"
tags: ["kubernetes"]
categories: ["kubernetes"]
---

# Kubernetes最佳实践：第一季概览

* 作者：Sandeep Dinesh,Google Developer Advocate

[原文](https://medium.com/google-cloud/kubernetes-best-practices-season-one-11119aee1d10)

Kubernetes很复杂，而且每天都变得越来越复杂。如果你是刚开始使用Kubernetes，或者已经在生产环境中运行了一段时间，那么很难跟上正在进行的快速开发步伐。当你有一个团队的人构建在kubernetes的时候，那么想要跟上其快速开发的步伐则显得更加艰难了，因为你必须保证团队里的每个人都保持更新且能同步到生产环境。

虽然市面上有大量的`hello world`经验的内容，但是使用kubernetes更多涉及到的是如何运行一个`Deployment`并通过`Service`把它暴露给外部世界。Kubernetes本身是白纸一张，基本上可以画任何你想画的东西上去，但是想知道从哪开始画却真的相当困难。

考虑到这一点，我开始根据我在日常与人交谈时收到的问题和反馈，开展题为“Kubernetes最佳实践”（以下是幻灯片和视频）的演讲。这个演讲非常受欢迎，我决定深入研究各个主题。我最初为这次演讲制作了七集内容（我想七集是非常合适的数量），我真的认为这些文章和视频可以帮助你和你的团队迅速提升Kubernetes。

所以这里是全部七集期待给你带来观赏乐趣的视频！ 我现在正在制作下一批视频，并希望得到您对想要观看的内容的反馈。欢迎发表评论或在推特上给我发消息，提出您的建议！

## 第一季

* [第一季全部视频](https://www.youtube.com/playlist?list=PLIivdWyY5sqL3xfXz5xJvwzFW_tlQB_GB)

### S01E01-如何以及为什么要构建尽量小的容器

![0*K0GIlb657-3AF8XC.jpg](https://cdn-images-1.medium.com/max/1600/0*K0GIlb657-3AF8XC.jpg)

在使用Kubernetes之前，你必须构建一些容器。Docker使得构建容器变得非常容易，但这也意味着很容易构建低效且不安全的容器。构建较小的容器可以很容易地提高Kubernetes集群的使用效率，而且这是一种不费力的方式。

* [博客](https://cloudplatform.googleblog.com/2018/04/Kubernetes-best-practices-how-and-why-to-build-small-container-images.html)
* [视频](https://youtu.be/wGz_cbtCiEA)

### S01E02-使用`namespace`来组织你的资源

![0*oE_d_44cAUSJ_wmg.jpg](https://cdn-images-1.medium.com/max/1600/0*oE_d_44cAUSJ_wmg.jpg)

一旦过了学习`Hello world`的阶段，在开始尝试管理在Kubernetes上运行的微服务时，您可能会遇到如何组织它们的问题。随着您的团队的成长并且您需要更多的可见性和控制权时，这会变得更糟。命名空间(namespace)提供了一种管理Kubernetes资源的强大方法，并为k8s的策略和管理提供了基础。

* [博客](https://cloudplatform.googleblog.com/2018/04/Kubernetes-best-practices-Organizing-with-Namespaces.html)
* [视频](https://youtu.be/xpnZX3if9Tc)

### S01E03-使用`readiness`和`liveness`探针来做健康检查

![0*rtpkUtv66zDCEUUX.jpg](https://cdn-images-1.medium.com/max/1600/0*rtpkUtv66zDCEUUX.jpg)

要创建强大可靠的服务需要支持健康检查。虽然Kubernetes具有默认的内置运行状况检查，但它们对于许多应用程序来说可能还不够。 `readiness`和`liveness`探针使您能够轻松地为您的应用程序自定义这些运行状况检查。

* [博客](https://cloudplatform.googleblog.com/2018/05/Kubernetes-best-practices-Setting-up-health-checks-with-readiness-and-liveness-probes.html)
* [视频](https://youtu.be/mxEvAPQRwhw)

### S01E04-资源请求和限制

![0*tiEgL6ffA_sqY7Ub.jpg](https://cdn-images-1.medium.com/max/1600/0*tiEgL6ffA_sqY7Ub.jpg)

内存泄漏，死循环，未知捣乱分子，过度配置，OMG，这种种问题！虽然Kubernetes为您提供了一个运行服务的强大平台，但如果您没有围绕资源定义规则，那么您最终将陷入困境。 但是值得庆幸的是，Kubernetes为您提供了大量的对资源及其使用方式的控制接口。

* [博客](https://cloudplatform.googleblog.com/2018/05/Kubernetes-best-practices-Resource-requests-and-limits.html)
* [视频](https://youtu.be/xjpHggHKm78)

### S01E05-优雅终止

![0*CkNz3GlnZGhyycfY.jpg](https://cdn-images-1.medium.com/max/1600/0*CkNz3GlnZGhyycfY.jpg)

Kubernetes中的Pod和Containers需要优雅地处理终止。 Kubernetes可以出于各种原因决定终止一个完全健康的Pod，并且干净地关闭是为您的用户提供良好体验的关键。

* [博客](https://cloudplatform.googleblog.com/2018/05/Kubernetes-best-practices-terminating-with-grace.html)
* [视频](https://youtu.be/Z_l_kE1MDTc)

### S01E06-映射外部服务

![0*BXOE0v4yChNM2i9s.jpg](https://cdn-images-1.medium.com/max/1600/0*BXOE0v4yChNM2i9s.jpg)

有可能您的服务存在于Kubernetes集群之外。其中一些可能是第三方服务，其他可能是您的团队或公司运行的服务。无论如何，生活在混合世界中会带来复杂性。Kubernetes能够映射这些外部服务，使其感觉就像是Kubernetes本地服务一样，从而更容易消除与外部服务一起工作的距离。

* [博客](https://cloudplatform.googleblog.com/2018/05/Kubernetes-best-practices-mapping-external-services.html)
* [视频](https://youtu.be/fvpq4jqtuZ8)

### S01E07-零停机升级集群

![0*UdMc07JcvGnbtudO.jpg](https://cdn-images-1.medium.com/max/1600/0*UdMc07JcvGnbtudO.jpg)

您需要做的最重要的事情之一是让您的群集保持最新状态。使用像GKE这样的托管服务可以使这更容易，但是你也可以使用一些其他方法使升级过程更加顺畅。

* [博客](https://cloudplatform.googleblog.com/2018/06/Kubernetes-best-practices-upgrading-your-clusters-with-zero-downtime.html)
* [视频](https://youtu.be/ajbC1yTW2x0)

从内容审阅到视频和博客编辑团队，感谢所有使这一切成为可能的人，使这个系列成为现实！
