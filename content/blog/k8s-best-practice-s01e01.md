---
title: "Kubernetes最佳实践S01E01：如何以及为什么构建尽量小的容器镜像"
date: 2018-08-10T14:26:18+08:00
draft: false
banner: "https://cdn-images-1.medium.com/max/1600/0*K0GIlb657-3AF8XC.jpg"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "今天是Google Developer Advocate `Sandeep Dinesh` 关于如何充分利用Kubernetes环境的七部分视频和博客系列的第一部分。 今天，他主要讲保持容器镜像尽可能小的理论和实用性。"
tags: ["kubernetes"]
categories: ["kubernetes"]
---

# Kubernetes最佳实践S01E01：如何以及为什么构建尽量小的容器镜像

* 作者：Sandeep Dinesh,Google Developer Advocate
* 日期：2018/04/20

[原文](https://cloudplatform.googleblog.com/2018/04/Kubernetes-best-practices-how-and-why-to-build-small-container-images.html)

**编者按**：今天是Google Developer Advocate `Sandeep Dinesh` 关于如何充分利用Kubernetes环境的七部分视频和博客系列的第一部分。 今天，他主要讲保持容器镜像尽可能小的理论和实用性。

Docker使构建容器镜像变得轻而易举。只需将标准 `Dockerfile` 放入您的文件夹，运行 `docker build` 命令，然后运行，芝麻开门！您的容器镜像已构建成功！

这种简单性的缺点是，很容易构建出大体积的容器镜像，其中包含您不需要的东西 - 包括潜在的安全漏洞。

在本期“Kubernetes最佳实践”中，让我们探讨如何使用`Alpine Linux` 和 `Docker builder` 模式创建生产就绪的容器镜像，然后做一些基准测试，然后确定这些容器在Kubernetes集群中运行方式。

根据您使用的是解释型语言还是编译型语言，创建容器镜像的过程会有所不同。让我们一起来深入了解！

## 解释型语言的容器化

解释型语言，如Ruby，Python，Node.js，PHP和其他语言通过发送源代码到解释器来运行代码。 这样的好处是可以跳过编译步骤，但其缺点是要求您将解释器与代码一起丢进去。

幸运的是，大多数这些语言都提供了预构建的Docker容器，其中包含一个轻量级环境，允许您运行更小的容器。

我们来看一个Node.js应用程序并对其进行容器化。 首先，让我们使用`node：onbuild`镜像作为基础。 Docker容器的`onbuild`版本预先打包了您需要的所有内容，因此无需执行大量配置即可搞定。 这意味着Dockerfile非常简单（只有两行！）。 但是你要付出的磁盘大小代价 - 差不多700MB！

```dockerfile
FROM node:onbuild
EXPOSE 8080
```

![1.png](https://4.bp.blogspot.com/-lweIsqWsBTU/WtjgagyqFBI/AAAAAAAAFcU/eSZU-2Ig1eIgJrZIeqhloRQe-Fr3Xcx0gCLcBGAs/s1600/1.png)

通过使用较小的基础镜像（如Alpine），您可以显著减少容器的大小。`Alpine Linux`是一款体积小，轻量级的Linux发行版，在Docker用户中非常受欢迎，因为它与许多应用程序兼容，同时仍然保持小体积。

幸运的是，Node.js（以及其他流行语言）有一个官方的`Alpine`图像，可以满足您的一切需求。与默认的`node`镜像不同，`node：alpine`会删除许多文件和程序，只留下足以运行您的应用程序的部分。

基于`Alpine Linux`的Dockerfile创建起来有点复杂，因为你必须运行一些针对`onbuild`的命令。

```dockerfile
FROM node:alpine
WORKDIR /app
COPY package.json /app/package.json
RUN npm install --production
COPY server.js /app/server.js
EXPOSE 8080
CMD npm start
```

但是，这是值得的，因为产生的镜像只有65MB！

![2.png](https://2.bp.blogspot.com/-Q2qeIhryIS4/WtjgU5OfUhI/AAAAAAAAFcQ/p9UWnKbCmnIeH1DCMyIjY_FSqiuJWNaFwCLcBGAs/s1600/2.png)

## 编译型语言的容器化

诸如Go，C，C ++，Rust，Haskell等编译型语言可以创建在没有许多外部依赖性的情况下运行的二进制文件。 这意味着你可以提前构建二进制文件并将其投入生产，而无需把创建二进制文件（如编译器）放进去。

借助Docker对[多阶段构建](https://docs.docker.com/engine/userguide/eng-image/multistage-build/)的支持，您可以轻松地打包二进制文件和最少量的脚手架。 让我们学习一下怎么做。

让我们采用Go应用程序并使用此模式对其进行容器化。 首先，让我们使用`golang：onbuild`镜像作为基础。 和以前一样，Dockerfile只有两行，但你再次付出磁盘大小超过700MB的代价！

```dockerfile
FROM golang:onbuild
EXPOSE 8080
```

![3.png](https://3.bp.blogspot.com/-fWp3NTbrQz8/WtjgPgn-DEI/AAAAAAAAFcI/Xfi8akc9Tro9n70Qf3fPOBPHKITcYZgnACLcBGAs/s1600/3.png)

下一步是使用更小的基础镜像，也就是`golang：alpine`镜像。 到目前为止，这与我们针对解释型语言所遵循的过程相同。

同样，使用Alpine基础镜像创建Dockerfile有点复杂，因为您必须运行一些执行`onbuild`镜像相关的命令。

```dockerfile
FROM golang:alpine
WORKDIR /app
ADD . /app
RUN cd /app && go build -o goapp
EXPOSE 8080
ENTRYPOINT ./goapp
```

但同样，由此产生的镜像要小得多，大小只有256MB！

![4.png](https://3.bp.blogspot.com/-mYXJQ8dfpRE/WtjgIHkjxDI/AAAAAAAAFcE/yUfnPtACXkI2slVHIZGVWdx2gK5CJ6LqgCLcBGAs/s1600/4.png)

但是，我们可以使镜像更小：您不需要Go附带的任何编译器或其他构建和调试工具，因此您可以从最终容器中删除它们。

让我们使用多阶段构建来获取由`golang：alpine`容器创建的二进制文件并将其自行打包。

```dockerfile
FROM golang:alpine AS build-env
WORKDIR /app
ADD . /app
RUN cd /app && go build -o goapp

FROM alpine
RUN apk update && \
   apk add ca-certificates && \
   update-ca-certificates && \
   rm -rf /var/cache/apk/*
WORKDIR /app
COPY --from=build-env /app/goapp /app
EXPOSE 8080
ENTRYPOINT ./goapp
```

你看看现在！ 这个容器镜像只有12MB大小！

![5.png](https://3.bp.blogspot.com/-QHfJW-5WNAU/Wtjf_DFyITI/AAAAAAAAFcA/fx9DsdmrzPMn3wB2Jdy9ZThZmaTCX-fOwCLcBGAs/s1600/5.png)

在构建此容器时，您可能会注意到Dockerfile会执行一些奇怪的操作，例如手动将HTTPS证书安装到容器中。这是因为基础的`Alpine Linux`几乎没有预安装任何东西。 因此，即使您需要手动安装任何依赖项，最终结果也还是超小容器镜像！

注意：如果您想节省更多空间，可以静态编译应用程序并使用`scratch`容器镜像。使用`scratch`作为基础容器镜像意味着您从头开始，根本没有基础层。 但是，我建议使用`Alpine`作为基础镜像而不是`scratch`，因为`Alpine`镜像中的仅仅增加少量额外MB的大小却可以使得使用标准工具和安装依赖项变得更加容易。

## 在哪构建和存储您的容器镜像？

为了构建和存储镜像，我强烈推荐`Google Container Builder`和`Google Container Registry`的组合。 `Google Container Builder`非常快，并自动将镜像推送到`Google Container Registry`。 大多数开发人员应该可以轻松地在GCP的免费套餐中完成所有工作，而`Google Container Registry`与原始Google云端存储的价格相同（便宜！）。

`Google Kubernetes Engine`等平台可以安全地从`Google Container Registry`中拉取镜像而无需任何其他配置，让您轻松上手！

此外，`Google Container Registry`还为您提供漏洞扫描工具和开箱即用的IAM支持。 这些工具可以使您更轻松地保护和锁定容器。

## 评估小体积容器的性能

人们声称小体积容器的巨大优势是缩短了时间 - 建立时间和拉取时间。 让我们测试一下，使用`onbuild`创建的容器，对比在多阶段过程中使用`Alpine`创建的容器，看看真实情况如何。

*TL;DR：对于功能强大的计算机或`Container Builder`没有显著差异，但对于性能没有那么强大的计算机和共享系统（如许多CI/CD系统）而言则有明显差异。 就绝对性能而言，小镜像总是更好。*

### 在大型机器上构建镜像

对于第一次测试，我将使用笔记本电脑进行构建。 我正在使用我们的办公室WiFi，所以下载速度非常快！

![6.png](https://4.bp.blogspot.com/-54jSQn0OHec/Wtjgrp7-rOI/AAAAAAAAFcg/m9wL0-BPhDY8VUoVJS0uIRFtybYQQrxkwCLcBGAs/s1600/6.png)

对于每次构建，我都会删除缓存中的所有Docker镜像。

##### 构建(Build)：

```
Go Onbuild: 35 Seconds
Go Multistage: 23 Seconds
```

对于较大的容器镜像，构建需要的时间大约耗费多10秒左右。 虽然这个代价仅在初始构建时需要付出，但是如果您使用持续集成系统，则在每次构建时都要付出这个代价。

下一个测试是将容器镜像推送到远程镜像仓库。 对于此测试，我使用`Google Container Registry`来存储镜像。

##### 推送(Push)：

```
Go Onbuild: 15 Seconds
Go Multistage: 14 Seconds
```

这很有趣！ 为什么需要花费相同的时间来推送12MB镜像和700MB镜像？ 事实证明，`Google Container Registry`在幕后使用了很多技巧，包括许多流行基础镜像的全局缓存。

最后，我想测试将镜像从远程镜像仓库库拉到本地计算机所需的时间。

##### 拉取(Pull)：

```
Go Onbuild: 26 Seconds
Go Multistage: 6 Seconds
```

20秒的差距，这是使用两个不同容器镜像之间的最大差异。 您可以开始看到使用较小镜像的优势，尤其是在经常拉取镜像时。

您还可以使用`Google Container Builder`在云中构建容器镜像，这样可以自动将它们存储在`Google Container Registry`中。

##### 构建(build)+推送(Push)

```
Go Onbuild: 25 Seconds
Go Multistage: 20 Seconds
```

再次证明，使用较小的镜像有一点小优势，没有我想象的那么大。

### 在小型机器上构建镜像

如果实在小型机器上构建镜像，那么使用较小的容器镜像是否有优势呢？ 如果你有一个功能强大的笔记本电脑与快速互联网连接和/或容器生成器，优势并不明显。 但是，如果您使用功能较弱的机器，故事就会发生变化。 为了模拟这一点，我使用了一个`Google Compute Engine f1-micro VM`来构建，推送和拉取这些镜像，结果非常惊人！

##### 构建(build)

```
Go Onbuild: 52 seconds
Go Multistage: 6 seconds
```

##### 推送(Push)

```
Go Onbuild: 54 seconds
Go Multistage: 28 seconds
```

##### 拉取(Pull)

```
Go Onbuild: 48 Seconds
Go Multistage: 16 seconds
```

在这种情况下，使用较小的容器镜像确实非常有帮助！

### 在kubernetes上拉取镜像

虽然您可能不关心构建和推送容器镜像所需的时间，但您应该非常关心拉取容器镜像所需的时间。 对于Kubernetes，这可能是您的生产集群最重要的指标。

例如，假设您有一个三节点集群，其中一个节点崩溃。 如果您使用的是像`Kubernetes Engine`这样的托管系统，系统会自动生成一个新节点来代替它。

但是，这个新节点将是全新的，并且必须先拉取所有容器镜像才能开始工作。 拉取容器镜像所需的时间越长，集群执行的时间就越长！

当您增加群集大小（例如，使用`Kubernetes Engine Autoscaling`）或将节点升级到新版本的Kubernetes时，可能会发生这种情况（接下来的几集中会关注这个问题）。

我们可以看到来自多个部署的多个容器的拉取性能可以在这里真正体现出来，使用小容器可能会浪费几分钟的部署时间！

## 安全性和漏洞

除了性能之外，使用较小的容器镜像还有很大的安全性提升。 与使用大基础镜像的容器镜像相比，小容器镜像通常具有较小的攻击面。

几个月前，我构建了`Go onbuild`和`multi-age`容器镜像，因此它们可能包含一些已被发现的漏洞。 使用`Google Container Registry`的内置漏洞扫描，可以轻松扫描容器中的已知漏洞。 让我们看看我们能找到些什么。

![7.png](https://1.bp.blogspot.com/-ov_9B_1zYLM/Wtje40i-0mI/AAAAAAAAFbo/RKqR6C-W8PUFeaxKmOuIstM7uzHI3IVGgCLcBGAs/s1600/7.png)

哇，这次两者之间有很大的不同！ 较小容器镜像中只有三个“中级”漏洞，而较大容器镜像中有16个严重漏洞和300多个其他漏洞。

让我们深入了解更大容器镜像里面存在的问题。

![8.png](https://1.bp.blogspot.com/-dZYGFWxicJs/WtjevtiBveI/AAAAAAAAFbk/wIxkM9w-kqYsof9apykSgKXn0-VojoEYwCLcBGAs/s1600/8.png)

您可以看到大多数存在问题的与我们的应用程序无关，甚至没有我们的应用程序！ 因为多阶段构建使用的是更小的基础镜像，所以可以产生漏洞的东西更少。

## 结论

使用小容器镜像的性能和安全优势不言而喻。使用小的基础镜像和`builder pattern`可以更容易地构建小镜像，并且还有许多其他技术可用于单个技术栈和编程语言，以最小化容器体积。 无论你做什么，你都可以确信你保持容器镜像最小化的努力是值得的！

[未完待续]第一季第二集中我们将讨论如何使用Kubernetes命名空间来隔离集群。 






















