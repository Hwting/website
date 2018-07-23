---
title: "七个构建容器的最佳实践(译)"
date: 2018-07-23T14:21:17+08:00
draft: false
banner: "https://1.bp.blogspot.com/-IXjF1_etXY0/Wz-6Qhob-cI/AAAAAAAAGBQ/2PUX2xrtXR0ydzzf4V8Aj4s0a8jHIeAlgCLcBGAs/s1600/gcp_kubernetes_docker_multi_stage_build_process.png"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "本文提供了有助于您有效构建容器的小技巧和最佳实践。"
tags: ["Docker"]
categories: ["Docker"]
---

# 七个构建容器的最佳实践(译)

[原文](https://cloudplatform.googleblog.com/2018/07/7-best-practices-for-building-containers.html)

# 七个构建容器的最佳实践

[Kubernetes Engine](https://cloud.google.com/kubernetes-engine/)是大规模运行工作负载的好地方。但在使用Kubernetes之前，您需要将应用程序容器化。您可以在Docker容器中运行大多数应用程序而不会有太多麻烦。但是，在生产中有效地运行这些容器并简化构建过程是另一回事。有许多事情需要注意，这样才能使您的安全团队和运营团队更加放心。本文提供了有助于您有效构建容器的小技巧和最佳实践。

### 1.每个容器只包含一个应用程序

[获取更多细节](https://cloud.google.com/solutions/best-practices-for-building-containers#package_a_single_application_per_container)

当只有一个应用程序在里面运行时，容器的效果最佳。此应用程序应具有单个父进程。例如，不要在同一容器中运行PHP和MySQL：更难调试，Linux信号将无法正确处理，无法水平扩展PHP容器等原因。这使您可以将应用程序的生命周期和该容器绑定在一起。

![gcp_kubernetes_container_best_practice.png](https://1.bp.blogspot.com/-2uX0pNjLVOY/Wz-582uzmiI/AAAAAAAAGBI/KiypCX0Gmh0jZ2lYXvkEToYRQESgPdm6ACLcBGAs/s1600/gcp_kubernetes_container_best_practice.png)

上图中左边的容器遵循最佳实践，右边的容器没有。

### 2.正确处理PID 1，信号处理和僵尸进程

[获取更多细节](https://cloud.google.com/solutions/best-practices-for-building-containers#signal-handling)

Kubernetes和Docker将Linux信号发送到容器内的应用程序以停止它，使用进程标识符（PID）1将这些信号发送到指定进程。如果您希望应用程序在需要时正常停止，则需要正确地处理这些信号。

谷歌开发者提倡的[Sandeep Dinesh](https://twitter.com/sandeepdinesh?lang=en)的文章-[Kubernetes最佳实践：如何优雅终止kubernetes](https://cloudplatform.googleblog.com/2018/05/Kubernetes-best-practices-terminating-with-grace.html) - 解释了整个Kubernetes终止生命周期。

### 3.优化Docker构建缓存

[获取更多细节](https://cloud.google.com/solutions/best-practices-for-building-containers#optimize-for-the-docker-build-cache)

Docker可以缓存镜像层以加速以后的构建。这是一个非常有用的功能，但在编写Dockerfiles时，需要介绍一下我们需要考虑的一些行为。 例如，您应该在Dockerfile中尽可能晚地添加应用程序的源代码，以便基本镜像和应用程序的依赖项得到缓存，并且不会在每次构建时重建。

以下面这个Dockerfile为例：

```dockerfile
FROM python:3.5
COPY my_code/ /src
RUN pip install my_requirements
```

你应该把最后两行交换一下：

```dockerfile
FROM python:3.5
RUN pip install my_requirements
COPY my_code/ /src
```

在新版本中，pip命令的结果将被缓存，并且每次源代码更改时都不会重新运行。

### 4.删除不必要的工具

[获取更多细节](https://cloud.google.com/solutions/best-practices-for-building-containers#remove_unnecessary_tools)

减少主机系统的攻击面总是一个好主意，而且使用容器比使用传统系统容易得多。从容器中删除应用程序不需要的所有内容。 或者更好的处理是，仅将您的应用程序包含在[distroless](https://github.com/GoogleContainerTools/distroless)或scratch镜像中。 如果可能，您还应该将容器的文件系统设置为只读。这样在您的绩效考核期间可以从安全团队获得一些出色的反馈。

### 5.尽可能地构建最小的镜像

[获取更多细节](https://cloud.google.com/solutions/best-practices-for-building-containers#build-the-smallest-image-possible)

谁喜欢下载数百兆的无用数据？大家都想获取最小的镜像。这样可以减少下载时间，冷启动时间和磁盘使用量。您可以使用多种策略来实现这一目标：从最小的基本镜像开始，利用镜像之间的公共层，并利用Docker的多阶段构建功能。

![gcp_kubernetes_docker_multi_stage_build_process.png](https://1.bp.blogspot.com/-IXjF1_etXY0/Wz-6Qhob-cI/AAAAAAAAGBQ/2PUX2xrtXR0ydzzf4V8Aj4s0a8jHIeAlgCLcBGAs/s1600/gcp_kubernetes_docker_multi_stage_build_process.png)

上图展示了Docker多阶段构建过程。

谷歌开发者提倡的[Sandeep Dinesh](https://twitter.com/sandeepdinesh?lang=en)的文章-Kubernetes最佳实践：[如何以及为何构建小型容器镜像](https://cloudplatform.googleblog.com/2018/04/Kubernetes-best-practices-how-and-why-to-build-small-container-images.html) - 深入介绍了这一主题。

### 6.正确地给镜像打标签

[获取更多细节](https://cloud.google.com/solutions/best-practices-for-building-containers#properly_tag_your_images)

标签可以帮助用户选择他们想要使用的镜像版本。给镜像打标签主要有两种方法：语义版本控制，或使用Git提交哈希。 无论您选择哪个，必须记录好，并清楚地符合用户的期望。 注意：虽然用户希望某些标签（如“latest”标签）从一个镜像移动到另一个镜像，但他们希望其他标签是不可变的，即使它们在技术上并非如此。 例如，一旦您使用“1.2.3”之类的标签标记了特定版本的镜像，就不应该再移动此标记。

### 7.仔细考虑是否使用公共镜像

[获取更多细节](https://cloud.google.com/solutions/best-practices-for-building-containers#carefully_consider_whether_to_use_a_public_image)

当你开始使用某个特定的软件时，使用公共镜像是个不错的选择。 但是，在生产中使用它们可能会带来一系列挑战，特别是在约束高的环境中。 例如，您可能需要控制其中的内容，或者您可能不想依赖外部存储库。 另一方面，为您使用的每个软件都构建自己的镜像并非易事，尤其是因为您需要跟上上游软件的安全更新。仔细权衡每种选择的优缺点，并做出明智的决定。

### 下一步

您可以阅读有关[容器构建最佳实践](https://cloud.google.com/solutions/best-practices-for-building-containers)的更多信息，和了解我们的[Kubernetes最佳实践](https://www.google.com/search?q=site%3Acloudplatform.googleblog.com%20%22kubernetes%20best%20practices%22)的更多信息。 您还可以试试我们的[Kubernetes Engine](https://cloud.google.com/kubernetes-engine/docs/quickstart)和[Container Builder](https://cloud.google.com/container-builder/docs/quickstarts)快速入门。




