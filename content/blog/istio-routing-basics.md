---
title: "Istio Routing 极简教程"
date: 2018-11-05T11:45:10+08:00
draft: false
banner: "https://cdn-images-1.medium.com/max/1600/1*WksXRI3XV54SegfmFd0_bg.png"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "在这篇文章中，我想介绍一下基础知识，并向您展示如何从头开始构建支持 Istio 的“HelloWorld”应用程序。 要记住的一点是，Istio 只管理您应用的流量。 在这种情况下，应用程序生命周期由底层平台 Kubernetes 管理。 因此，您需要了解容器和 Kubernetes 基础知识，并且需要了解 Istio Routing 原语，例如 Gateway，VirtualService，DestinationRule。 我假设大多数人都知道容器和 Kubernetes 基础知识。 我将在本文中专注于 Istio Routing。"
tags: ["Istio"]
categories: ["Istio"]
---

# Istio Routing 极简教程

[原文](https://medium.com/google-cloud/istio-routing-basics-14feab3c040e)

在学习像 Istio 这样的新技术时，看一下示例应用程序总是一个好主意。 Istio repo 有一些示例应用程序，但它们似乎有各种不足。 文档中的 BookInfo 是一个很好的示例。 但是，对于我而言，它太冗长，服务太多，而且文档似乎专注于管理 BookInfo 应用程序，而不是从头开始构建。 有一个较小的 helloworld 例子，但它更多的是关于自动伸缩而不是其他。

在这篇文章中，我想介绍一下基础知识，并向您展示如何从头开始构建支持 Istio 的“HelloWorld”应用程序。 要记住的一点是，Istio 只管理您应用的流量。 在这种情况下，应用程序生命周期由底层平台 Kubernetes 管理。 因此，您需要了解容器和 Kubernetes 基础知识，并且需要了解 Istio Routing 原语，例如 Gateway，VirtualService，DestinationRule。 我假设大多数人都知道容器和 Kubernetes 基础知识。 我将在本文中专注于 Istio Routing。

## 基础步骤

以下这些大致就是您需要遵循的，以获得 Istio 的“HelloWorld”应用程序的步骤：

1. 创建一个 Kubernetes 集群并安装带有 sidecare 自动注入的 Istio。
2. 使用您选择的语言创建 Hello World 应用程序，创建 Docker 镜像并将其推送到公共镜像仓库。
3. 为你的容器创建 Kubernetes deployment 和 service。
4. 创建 Gateway 以启用到群集的 HTTP(S)流量。
5. 创建 VirtualService,通过 Gateway 公开 Kubernetes 服务。
6. （可选）如果要创建多个版本应用程序，请创建 DestinationRule 以定义可从 VirtualService 引用的 subsets。
7. （可选）如果要在服务网格外部调用其他外部服务，请创建 ServiceEntry。

我不会在本文中介绍步骤 1 和 2，因为它们不是特定于 Istio 的。 如果您需要有关这些步骤的帮助，可以查看我在本文末尾提到的文章。 第 3 步也不是 Istio 特定的，但它是其他一切的先决条件，所以让我们从那开始。

## Deployment 和 Service

正如我所提到的，应用程序生命周期由 Kubernetes 管理。 因此，您需要从创建 Kubernetes deployment 和 service 开始。 我的情况如下，我有一个容器化的 ASP.NET 核心应用程序，其镜像我已经推送到谷歌镜像仓库。 让我们从创建一个`aspnetcore.yaml`文件开始：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: aspnetcore-service
  labels:
    app: aspnetcore
spec:
  ports:
    - port: 8080
      name: http
  selector:
    app: aspnetcore
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: aspnetcore-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: aspnetcore
        version: v1
    spec:
      containers:
        - name: aspnetcore
          image: gcr.io/istio-project-212517/hello-dotnet:v1
          imagePullPolicy: Always #IfNotPresent
          ports:
            - containerPort: 8080
```

创建 Deployment 和 Service：

```shell
$ kubectl apply -f aspnetcore.yaml
service "aspnetcore-service" created
deployment.extensions "aspnetcore-v1" created
```

到目前为止没有任何特定的针对 Istio 的内容。

## Gateway

我们现在可以开始研究 Istio Routing。 首先，我们需要为服务网格启用 HTTP/HTTPS 流量。 为此，我们需要创建一个网关。 Gateway 描述了在边缘运行的负载均衡，用于接收传入或传出的 HTTP/TCP 连接。

让我们创建一个`aspnetcore-gateway.yaml`文件:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: aspnetcore-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
```

创建 Gateway：

```shell
$ kubectl apply -f aspnetcore-gateway.yaml
gateway.networking.istio.io "aspnetcore-gateway" created
```

此时，我们为集群启用了 HTTP 流量。 我们需要将之前创建的 Kubernetes 服务映射到 Gateway。 我们将使用 VirtualService 执行此操作。

## VirtualService

VirtualService 实际上将 Kubernetes 服务连接到 Istio 网关。 它还可以执行更多操作，例如定义一组流量路由规则，以便在主机被寻址时应用，但我们不会深入了解这些细节。

让我们创建一个`aspnetcore-virtualservice.yaml`文件:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: aspnetcore-virtualservice
spec:
  hosts:
    - "*"
  gateways:
    - aspnetcore-gateway
  http:
    - route:
        - destination:
            host: aspnetcore-service
```

请注意，VirtualService 与特定网关绑定，并定义引用 Kubernetes 服务的主机。

创建 VirtualService：

```shell
$ kubectl apply -f aspnetcore-virtualservice.yaml
virtualservice.networking.istio.io "aspnetcore-virtualservice" created
```

## 测试 V1 版本 APP

我们准备测试我们的应用程序了。 我们需要获取 Istio Ingress Gateway 的 IP 地址：

```shell
$ kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP
istio-ingressgateway   LoadBalancer   10.31.247.41   35.240.XX.XXX
```

当我们在浏览器中打开`EXTERNAL-IP`时，我们应该看到 HelloWorld ASP.NET Core 应用程序：

![1*WksXRI3XV54SegfmFd0_bg.png](https://cdn-images-1.medium.com/max/1600/1*WksXRI3XV54SegfmFd0_bg.png)

## DestinationRule

在某些时候，您希望将应用更新为新版本。 也许你想分割两个版本之间的流量。 您需要创建一个 DestinationRule 来定义是哪些版本，在 Istio 中称为 subset。

首先，更新 aspnetcore.yaml 文件以使用 v2 版本的容器定义 v2 的 deployment：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: aspnetcore-service
  labels:
    app: aspnetcore
spec:
  ports:
    - port: 8080
      name: http
  selector:
    app: aspnetcore
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: aspnetcore-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: aspnetcore
        version: v1
    spec:
      containers:
        - name: aspnetcore
          image: gcr.io/istio-project-212517/hello-dotnet:v1
          imagePullPolicy: Always #IfNotPresent
          ports:
            - containerPort: 8080
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: aspnetcore-v2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: aspnetcore
        version: v2
    spec:
      containers:
        - name: aspnetcore
          image: gcr.io/istio-project-212517/hello-dotnet:v2
          imagePullPolicy: Always #IfNotPresent
          ports:
            - containerPort: 8080
```

创建新的 Deployment:

```shell
$ kubectl apply -f aspnetcore.yaml
service "aspnetcore-service" unchanged
deployment.extensions "aspnetcore-v1" unchanged
deployment.extensions "aspnetcore-v2" created
```

如果使用 EXTERNAL-IP 刷新浏览器，您将看到应用程序的 v1 和 v2 版本交替出现：

![1*WksXRI3XV54SegfmFd0_bg.png](https://cdn-images-1.medium.com/max/1600/1*WksXRI3XV54SegfmFd0_bg.png)

![1*vqfpTeJtV1aP8v0mV8Ff2g.png](https://cdn-images-1.medium.com/max/1600/1*vqfpTeJtV1aP8v0mV8Ff2g.png)

这是符合预期的，因为两个版本都暴露在相同的 Kubernetes 服务之后：aspnetcore-service。

如果您想将服务仅指向 v2，该怎么办？ 这可以通过在 VirtualService 中指定 subset 来完成，但我们需要首先在 DestinationRules 中定义这些 subset。 DestinationRule 本质上是将标签映射到 Istio 的 subset。

创建一个`aspnetcore-destinationrule.yaml`文件：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: aspnetcore-destinationrule
spec:
  host: aspnetcore-service
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

创建 DestinationRule：

```shell
$ kubectl apply -f aspnetcore-destinationrule.yaml
destinationrule.networking.istio.io "aspnetcore-destinationrule" created
```

现在你可以从 VirtualService 来引用 v2 subset：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: aspnetcore-virtualservice
spec:
  hosts:
  - "*"
  gateways:
  - aspnetcore-gateway
  http:
  - route:
    - destination:
        host: aspnetcore-service
         subset: v2
```

更新 VirtualService:

```shell
$ kubectl apply -f aspnetcore-virtualservice.yaml
virtualservice.networking.istio.io "aspnetcore-virtualservice" configured
```

如果您现在继续浏览 EXTERNAL-IP，您现在应该只能看到应用程序的 v2 版本。

## ServiceEntry

我想在 Istio Routing 中提到的最后一件事是 ServiceEntry。 默认情况下，Istio 中的所有外部流量都被阻止。 如果要启用外部流量，则需要创建 ServiceEntry 以列出为外部流量启用的协议和主机。 我不会在这篇文章中展示一个例子，但你可以在[这里](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#ServiceEntry)阅读更多相关内容。

希望这篇文章对你有用！ 如果您想了解更多信息，可以使用 codelab 系列以下两部分，其中所有这些概念和更多内容将在逐步的详细教程中进行说明：

- [Deploy ASP.NET Core app to Google Kubernetes Engine with Istio (Part 1)](https://codelabs.developers.google.com/codelabs/cloud-istio-aspnetcore-part1)

- [Deploy ASP.NET Core app to Google Kubernetes Engine with Istio (Part 2)](https://codelabs.developers.google.com/codelabs/cloud-istio-aspnetcore-part2)
