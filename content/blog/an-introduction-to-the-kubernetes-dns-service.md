---
title: "Kubernetes DNS 服务简介"
date: 2018-10-31T09:54:07+08:00
draft: false
banner: "https://community-cdn-digitalocean-com.global.ssl.fastly.net/assets/tutorials/images/large/kubernetes_tutorials.png?1539969788"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "最近版本的 Kubernetes 中 Kubernetes DNS 服务的实现细节已经改变。 在本文中，我们将介绍 Kubernetes DNS 服务的 kube-dns 和 CoreDNS 几个不同的实现版本。 我们一起来看看它们的运作方式以及 Kubernetes 生成的 DNS 记录。"
tags: ["Kubernetes"]
categories: ["Kubernetes"]
---

# Kubernetes DNS 服务简介

[原文](https://www.digitalocean.com/community/tutorials/an-introduction-to-the-kubernetes-dns-service)

## 介绍

域名系统（DNS）是一种用于将各种类型的信息（例如 IP 地址）与易于记忆的名称相关联的系统。 默认情况下，大多数 Kubernetes 群集会自动配置内部 DNS 服务，以便为服务发现提供轻量级机制。 内置的服务发现使应用程序更容易在 Kubernetes 集群上相互查找和通信，即使在节点之间创建，删除和移动 pod 和服务时也是如此。

最近版本的 Kubernetes 中 Kubernetes DNS 服务的实现细节已经改变。 在本文中，我们将介绍 Kubernetes DNS 服务的 kube-dns 和 CoreDNS 几个不同的实现版本。 我们一起来看看它们的运作方式以及 Kubernetes 生成的 DNS 记录。

如果你想在开始之前就更全面地了解 DNS，请阅读“[DNS 术语，组件和概念简介](https://www.digitalocean.com/community/tutorials/an-introduction-to-dns-terminology-components-and-concepts)”。 对于您可能不熟悉的任何 Kubernetes 主题，您可以阅读“[Kubernetes 简介](https://www.digitalocean.com/community/tutorials/an-introduction-to-kubernetes)。”

## Kubernetes DNS 服务提供什么？

在 Kubernetes 版本 1.11 之前，Kubernetes DNS 服务基于 kube-dns。 1.11 版引入了 CoreDNS 来解决 kube-dns 的一些安全性和稳定性问题。

无论处理实际 DNS 记录的软件如何，两种实现都以类似的方式工作：

- 创建名为 kube-dns 的服务和一个或多个 pod。
- kube-dns 服务监听来自 Kubernetes API 的服务 service 和端点 endpoint 事件，并根据需要更新其 DNS 记录。 创建，更新或删除 Kubernetes 服务及其关联的 pod 时会触发这些事件。
- kubelet 将每个新 pod 的/etc/resolv.conf 名称服务器选项设置为 kube-dns 服务的集群 IP，并使用适当的搜索选项以允许使用更短的主机名：

resolve.conf

```
nameserver 10.32.0.10
search namespace.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

- 然后，在容器中运行的应用程序可以将主机名（例如 example-service.namespace）解析为正确的群集 IP 地址。

## Kubernetes DNS 记录示例

Kubernetes 服务的完整 DNS A 记录类似于以下示例：

```
service.namespace.svc.cluster.local
```

一个 pod 会有这种格式的记录，反映了 pod 的实际 IP 地址：

```
10.32.0.125.namespace.pod.cluster.local
```

此外，为 Kubernetes 服务的命名端口创建 SRV 记录：

```
_port-name._protocol.service.namespace.svc.cluster.local
```

所有这些的结果是内置的，基于 DNS 的服务发现机制，您的应用程序或微服务可以在其中定位一个简单一致的主机名，并可以访问群集上的其他服务或 pod。

## 搜索域和解析较短的主机名

由于`resolv.conf`文件中列出了搜索域后缀，所以您通常不需要使用完整主机名来访问其他服务。 如果要在同一命名空间中寻址其它服务，则只需使用服务名称即可：

```
other-service
```

如果服务位于不同的命名空间中，请将其添加：

```
other-service.other-namespace
```

如果您要定位 pod，则至少需要像以下所示使用：

```
pod-ip.other-namespace.pod
```

正如我们在默认的 resolv.conf 文件中看到的那样，只有.svc 后缀会自动完成，因此请确保指定.pod 之前的所有内容。

现在我们已经了解了 Kubernetes DNS 服务的实际用途，让我们来看看两个不同实现的一些细节。

## Kubernetes DNS 实现细节

如上一节所述，Kubernetes 1.11 版引入了处理 kube-dns 服务的新版本 CoreDNS。 这样做的动机是提高服务的性能和安全性。 我们先来看看原始的 kube-dns 实现。

### kube-dns

Kubernetes 1.11 之前的 kube-dns 服务由在 kube-system 命名空间中的 kube-dns pod 中运行的三个容器组成。 这三个容器是：

- kube-dns：运行 SkyDNS 的容器，用于执行 DNS 查询解析
- dnsmasq：流行的轻量级 DNS 解析器和缓存，用于缓存 SkyDNS 的响应
- sidecar：一个 sidecar 容器，用于处理指标报告并响应服务的运行状况检查

Dnsmasq 中的安全漏洞以及 SkyDNS 的扩展性能问题导致了被 CoreDNS 所替换。

### CoreDNS

从 Kubernetes 1.11 开始，新的 Kubernetes DNS 服务，CoreDNS 已升级为 GA。 这意味着它已准备好用于生产，并且将成为许多安装工具和托管 Kubernetes 提供商的默认集群 DNS 服务。

CoreDNS，用 Go 编写且单一进程，它涵盖了以前系统的所有功能： 单个容器解析并缓存 DNS 查询，响应运行状况检查并提供指标。

除了解决与性能和安全相关的问题之外，CoreDNS 还修复了一些其他小错误并添加了一些新功能：

- 修复了使用 stubDomains 和外部服务之间不兼容的一些问题
- CoreDNS 可以通过随机化返回某些记录的顺序来增强基于 DNS 的 round-robin 负载平衡
- autopath 功能可以在解析外部主机名时提高 DNS 响应时间，方法是更好地遍历 resolv.conf 中列出的每个搜索域后缀
- 如果使用 kube-dns 的话，10.32.0.125.namespace.pod.cluster.local 将始终解析为 10.32.0.125，即使 pod 实际上不存在。 CoreDNS 具有“已验证的 pod”模式，只有当存在具有正确 IP 且位于右侧命名空间的 pod 时，才会成功解析。

有关 CoreDNS 及其与 kube-dns 的不同之处的更多信息，您可以阅读[Kubernetes CoreDNS GA 公告](https://kubernetes.io/blog/2018/07/10/coredns-ga-for-kubernetes-cluster-dns/)。

## 其他配置选项

kubernetes 运营商通常希望自定义其 pod 和容器如何解析某些自定义域，或者需要调整上游名称服务器或搜索 resolv.conf 中配置的域后缀。 您可以使用 pod 规范的 dnsConfig 选项执行此操作：

example_pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: example
  name: custom-dns
spec:
  containers:
    - name: example
      image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 203.0.113.44
    searches:
      - custom.dns.local
```

更新此配置将重写 pod 的 resolv.conf 以启用更改。 配置直接映射到标准的 resolv.conf 选项，因此上面的配置将新增`nameserver 203.0.113.44`和`search custom.dns.local`这几行。

## 结论

在本文中，我们介绍了 Kubernetes DNS 为开发人员提供了哪些服务的基础知识，展示了 service 和 pod 的一些 DNS 记录示例，讨论了不同的 Kubernetes 版本上的不同实现，并突出显示了一些可用于自定义 pod 解析 DNS 查询的其他配置选项。
