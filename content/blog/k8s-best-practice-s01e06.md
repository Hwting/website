---
title: "K8s Best Practice S01e06"
date: 2018-08-18T19:09:26+08:00
draft: true
banner: "https://cdn-images-1.medium.com/max/1600/0*BXOE0v4yChNM2i9s.jpg"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "今天是Google Developer Advocate `Sandeep Dinesh`关于如何充分利用Kubernetes环境的七部分视频和博客系列的第六部分。"
tags: ["kubernetes"]
categories: ["kubernetes"]
---

# kubernetes最佳实践S01E06：映射外部服务

* 作者：Sandeep Dinesh, Google Developer Advocate
* 日期：2018/05/25

[原文](https://cloudplatform.googleblog.com/2018/05/Kubernetes-best-practices-mapping-external-services.html)

编者按：今天是Google Developer Advocate `Sandeep Dinesh`关于如何充分利用Kubernetes环境的七部分视频和博客系列的第六部分。

如果您像大多数Kubernetes用户一样，您可能会使用群集外的服务。 例如，您可能使用Twillio API发送短信，或者使用Google Cloud Vision API进行图像分析。

如果您的不同环境中的应用程序连接到同一外部端点，并且没有计划将外部服务引入Kubernetes集群，则可以直接在代码中使用外部服务端点。 但是，在许多情况下情况并非如此。

一个很好的例子是数据库。 虽然某些云原生数据库（如Cloud Firestore或Cloud Spanner）使用单个端点进行所有访问，但大多数数据库都有针对不同实例的单独端点。

此时，您可能认为找到端点的一个好方法是使用ConfigMaps。 只需将端点地址存储在ConfigMap中，并在代码中将其用作环境变量。 虽然这种解决方案有效，但也存在一些缺点。 您需要修改部署以包含ConfigMap并编写其他代码以从环境变量中读取。 但最重要的是，如果端点地址发生更改，则可能需要重新启动所有正在运行的容器以获取更新的端点地址。

在本期“Kubernetes最佳实践”中，让我们学习如何利用Kubernetes的内置服务发现机制来运行集群外部的服务，就像集群内的服务一样！ 这使您可以在dev和prod环境中进行校验，最终只需要在集群中迁移服务，而根本不必更改代码。

## 场景1：具有IP地址的群集外的数据库

一种非常常见的情况是您托管自己的数据库，但在群集外部执行此操作，例如在Google Compute Engine实例上。 如果您在Kubernetes内部和外部运行某些服务，或者需要比Kubernetes允许的更多自定义或控制，这是非常常见的。

您希望在某些时候，可以迁移集群内的所有服务，但在此之前，您将生活在混合世界中。 值得庆幸的是，您可以使用静态Kubernetes服务来缓解一些痛苦。

在此示例中，我使用Cloud Launcher创建了一个MongoDB服务器。 由于它是在与Kubernetes集群相同的网络（或VPC）中创建的，因此可以使用高性能内部IP地址进行访问。 在Google Cloud中，这是默认设置，因此您无需任何特殊配置。

![gcp-kubernetes-internal-ip-address.png](https://4.bp.blogspot.com/-r5AWmuJhv5g/WwYC6NJQu_I/AAAAAAAAFu0/_b5utAQQ13kiQlHL7fzUy0ZgG4V34wJqACLcBGAs/s1600/gcp-kubernetes-internal-ip-address.png)

现在我们有了IP地址，第一步是创建服务：

```yaml
kind: Service
apiVersion: v1
metadata:
 name: mongo
Spec:
 type: ClusterIP
 ports:
 - port: 27017
   targetPort: 27017
```

您可能会注意到此服务没有Pod选择器。 这会创建一个服务，但它不知道在哪里发送流量。 这允许您手动创建将从此服务接收流量的Endpoints对象。

```yaml
kind: Endpoints
apiVersion: v1
metadata:
 name: mongo
subsets:
 - addresses:
     - ip: 10.240.0.4
   ports:
     - port: 27017
```

您可以看到端点手动定义数据库的IP地址，并使用与服务相同的名称。 Kubernetes使用端点中定义的所有IP地址，就像它们是常规的Kubernetes Pod一样。 现在，您可以使用简单的连接字符串访问数据库：

```
mongodb://mongo
```

> 根本不需要在代码中使用IP地址！ 如果将来IP地址发生变化，您可以使用新IP地址更新端点，并且您的应用程序无需进行任何更改。

## 场景2：具有URI的远程托管数据库

如果您使用来自第三方的托管数据库服务，则可能会为您提供可用于连接的统一资源标识符（URI）。 如果他们为您提供IP地址，则可以使用方案1中的方法。

在这个例子中，我有两个在mLab上托管的MongoDB数据库。 其中一个是我的开发数据库，另一个是生产。

![gcp-kubernetes-mongodb-deployments.png](https://3.bp.blogspot.com/-_RRGLSWY1Tg/WwYC6KDU0iI/AAAAAAAAFu8/QhN6S0PuL28nTFrXqitzfdyGnixLXtj3QCEwYBhgL/s1600/gcp-kubernetes-mongodb-deployments.png)

这些数据库的连接字符串如下：

```
mongodb://<dbuser>:<dbpassword>@ds149763.mlab.com:49763/dev
```

```
mongodb://<dbuser>:<dbpassword>@ds145868.mlab.com:45868/prod
```

mLab为您提供动态URI和动态端口，您可以看到它们都是不同的。 让我们使用Kubernetes为这些差异创建一个抽象层。 在这个例子中，让我们连接到dev数据库。

您可以创建一个叫“ExternalName”的Kubernetes服务，它为您提供静态Kubernetes服务，将流量重定向到外部服务。 此服务在内核级别执行简单的CNAME重定向，因此对性能的影响非常小。

该服务的YAML如下所示：

```yaml
kind: Service
apiVersion: v1
metadata:
 name: mongo
spec:
 type: ExternalName
 externalName: ds149763.mlab.com
```

现在，您可以使用更简化的连接字符串：

```
mongodb://<dbuser>:<dbpassword>@mongo:<port>/dev
```

由于“ExternalName”使用CNAME重定向，因此无法进行端口重新映射。 对于具有静态端口的服务，这可能没问题，但不幸的是，在示例中它的端口是动态的。 mLab的免费套餐为您提供动态端口号，您无法更改它。 这意味着您需要为dev和prod使用不同的连接字符串。

但是，如果你可以获得IP地址，那么你可以进行端口重映射，我将在下一节中解释。

## 场景3：具有URI和端口重新映射的远程托管数据库

虽然CNAME重定向适用于每个环境具有相同端口的服务，但在每个环境的不同端点使用不同端口的情况下，它略显不足。 谢天谢地，我们可以使用一些基本工具解决这个问题。

第一步是从URI获取IP地址。

如果对URI运行nslookup，hostname或ping命令，则可以获取数据库的IP地址。

![gcp-kubernetes-nslookup.png](https://2.bp.blogspot.com/-LAq32CiKAYc/WwYC6LlreYI/AAAAAAAAFvE/9_-BMf0k21Qt9PLsz_9vwLOogZ4JNBgeQCEwYBhgL/s1600/gcp-kubernetes-nslookup.png)

您现在可以创建重新映射mLab端口的服务以及此IP地址的端点。

```yaml
kind: Service
apiVersion: v1
metadata:
 name: mongo
spec:
 ports:
 - port: 27017
   targetPort: 49763
---
kind: Endpoints
apiVersion: v1
metadata:
 name: mongo
subsets:
 - addresses:
     - ip: 35.188.8.12
   ports:
     - port: 49763
```

*注意：URI可能使用DNS对多个IP地址进行负载均衡，因此如果IP地址发生变化，此方法可能存在风险！ 如果从上面的命令中获得多个IP地址，则可以将所有这些地址包含在端点YAML中，并且Kubernetes将对所有IP地址的流量进行负载均衡。*

这样，您无需指定端口即可连接到远程数据库。 Kubernetes服务透明地重新映射端口！

```
mongodb://<dbuser>:<dbpassword>@mongo/dev
```

## 结论

将外部服务映射成内部服务使您可以灵活地将这些服务引入集群，同时最大限度地减少重构工作。 即使你今天不计划这么做，那你也永远不知道明天会发生什么！ 此外，它还可以更轻松地管理和了解您的组织正在使用哪些外部服务。

如果外部服务具有有效的域名并且您不需要端口重新映射，则使用“ExternalName”服务类型是将外部服务映射到内部服务的简单快捷的方法。 如果您没有域名或需要进行端口重新映射，只需将IP地址添加到端点并使用它。