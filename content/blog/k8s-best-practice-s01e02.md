---
title: "kubernetes最佳实践S01E02：使用命名空间管理资源"
date: 2018-08-12T22:19:22+08:00
draft: false
banner: "https://cdn-images-1.medium.com/max/1600/0*oE_d_44cAUSJ_wmg.jpg"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "今天是Google Developer Advocate `Sandeep Dinesh`的七部分视频和博客系列的第二部分，介绍如何充分利用您的Kubernetes环境。"
tags: ["kubernetes"]
categories: ["kubernetes"]
---

# kubernetes最佳实践S01E02：使用命名空间管理资源

* 作者：Sandeep Dinesh, Google Developer Advocate
* 日期：2018/04/27

[原文](https://cloudplatform.googleblog.com/2018/04/Kubernetes-best-practices-Organizing-with-Namespaces.html)

编者注：今天是Google Developer Advocate `Sandeep Dinesh`的七部分视频和博客系列的第二部分，介绍如何充分利用您的Kubernetes环境。

当您开始在Kubernetes之上构建越来越多的服务时，简单的任务开始变得更加复杂。 例如，团队无法创建具有相同名称的Kubernetes `Service`或`Deployment`。 如果你有成千上万的`Pod`，只是列出它们都需要一些时间，更不用说实际管理它们了!当然，这些还只是冰山一角。

在本期Kubernetes最佳实践中，让我们来看看如何使用Kubernetes命名空间来更轻松地管理您的Kubernetes资源。

## 什么是命名空间(Namespace)?

您可以将命名空间视为Kubernetes集群中的虚拟集群。 您可以在单个Kubernetes集群中拥有多个名称空间，并且它们在逻辑上彼此隔离。 他们可以为您和您的团队提供组织，安全甚至性能方面的帮助！

## 默认命名空间(Default Namespace)

在大多数Kubernetes发行版中，集群开箱即用，名称空间默认为`default`。事实上，Kubernetes实际上有三个名称空间：`default`，`kube-system`（用于Kubernetes组件）和`kube-public`（ 用于公共资源）。 `kube-public`现在并没有真正使用过，而且通常单独隔离一个`kube-system`是个好主意，尤其是在`Google Kubernetes Engine`这样的托管系统中。 默认名称空间是你创建服务和应用程序的默认位置，如果你不指定`namespace`参数的话。

这个命名空间绝对没有什么特别之处，除了Kubernetes工具是开箱即用的设置使用这个命名空间外，而且你无法删除它。 虽然它很适合入门和小型生产系统，但我建议不要在大型生产系统中使用它。 这是因为团队很容易在没有意识到的情况下，意外地覆盖或破坏其他服务。 相反，我们应该创建多个名称空间并使用它们来将服务划分为可管理的块。

## 创建命名空间

不要害怕创建名称空间。 它们不会增加性能损失，而且实际上，在许多情况下它们可以提高性能，因为这样的话`Kubernetes API`使用的是较小的对象集合。

可以使用单个命令来创建命名空间。 如果你想创建一个名为`test`的命名空间，你可以运行如下命令：

```shell
kubectl create namespace test
```

或者您可以像创建其他任何Kubernetes资源一样，创建一个`YAML`文件并应用它。

`test.yaml`:

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: test
  labels:
    name: test
```

```shell
kubectl apply -f test.yaml
```

## 查看命名空间

您可以使用以下命令查看所有命名空间：

```shell
kubectl get namespace
```

![kubectl-get-namespace-google-cloud-platform.png](https://1.bp.blogspot.com/-7ufbzm6MrQk/WuDCvMbwbLI/AAAAAAAAFdw/2AloDGeLGCwap_18dSV8XjzjrvWgSmb3QCLcBGAs/s1600/kubectl-get-namespace-google-cloud-platform.png)

如上图，您可以看到三个内置命名空间，以及名为`test`的新命名空间。

## 在命名空间中创建资源

让我们看一个简单的YAML,它用来创建一个Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  labels:
    name: mypod
spec:
  containers:
  - name: mypod
    image: nginx
```

您可能会注意到在任何地方都没有提到名称空间。 如果在此文件上运行`kubectl apply`，它将在当前活动的命名空间中创建Pod。 除非您更改它，否则这将是“默认”命名空间。

有两种方法可以明确告诉Kubernetes您要在哪个`Namespace`中创建资源。

一种方法是在创建资源时设置`namespace`标识：

```shell
kubectl apply -f pod.yaml --namespace=test
```

您还可以在YAML声明中指定命名空间:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  namespace: test
  labels:
    name: mypod
spec:
  containers:
  - name: mypod
    image: nginx
```

如果在YAML声明中指定命名空间，则将始终在该命名空间中创建资源。 如果您尝试使用`namespace`标志来设置另一个命名空间，则该命令将会失败。

## 在命名空间中查看资源

如果你试图找到你的Pod，你可能会注意到你不能！

```
$ kubectl get pods
No resources found.
```

这是因为所有命令都是针对当前active的命名空间运行的。 要查找Pod，您需要指定`namespace`。

```
$ kubectl get pods --namespace=test
NAME      READY     STATUS    RESTARTS   AGE
mypod     1/1       Running   0          10s
```

这可能会很快让人觉得很烦，特别是如果您是一个开发团队的开发人员，该团队使用自己的命名空间来处理所有事情，并且不希望对每个命令都指定`namespace`。 让我们看看我们如何解决这个问题。

## 管理active命名空间

初始状态下，您的活动命名空间是`default`。 除非在YAML中指定命名空间，否则所有Kubernetes命令都将使用当前active命名空间。

不幸的是，尝试使用kubectl管理您的active命名空间可能会很痛苦。 幸运的是，有一个非常好的工具叫做`kubens`（由优秀的Ahmet Alp Balkan创建）可以让它变得轻而易举！

运行`kubens`命令时，您应该看到所有命名空间，并突出显示active命名空间：

![kubens-command-google-cloud-platform.png](https://3.bp.blogspot.com/-XIgOnHpQNcA/WuDDf1N8PYI/AAAAAAAAFd4/GfM2Kw-jz6I_v8wtIy6iVClAVa7atbl5ACLcBGAs/s1600/kubens-command-google-cloud-platform.png)

要将active命名空间切换到`test`命名空间，请运行：

```shell
kubens test
```

现在您可以看到`test`命名空间处于active状态：

![kubens-test-namespace-google-clou-platform.png](https://1.bp.blogspot.com/-74riYpXxk84/WuDDoLJWRRI/AAAAAAAAFd8/k2NHenzJy383SPPl1JpYS3qIII_Q-d3jQCLcBGAs/s1600/kubens-test-namespace-google-clou-platform.png)

现在，如果你运行kubectl命令，命名空间将是`test`而不是`default`！ 
这意味着您不需要指定命名空间来查看测试命名空间中的pod。

```
$ kubectl get pods
NAME      READY     STATUS    RESTARTS   AGE
mypod     1/1       Running   0          10m
```

## 跨命名空间通信

命名空间彼此“隐藏”，但默认情况下它们不是完全隔离的。 一个命名空间中的服务可以与另一个命名空间中的服务进行通信。 这通常非常有用，例如让您的团队的服务（在您的命名空间中）与另一个团队的服务（在另一个命名空间中）进行通信。

当您的应用想要访问Kubernetes `Service`时，您可以使用内置的DNS服务发现，只需将您的应用指向该`Service`的名称即可。 但是，您可以在多个命名空间中创建具有相同名称的`Service`！ 值得庆幸的是，通过使用扩展形式的DNS地址很容易解决这个问题。

Kubernetes中的服务使用通用DNS模式公开其`endpoint`。 它看起来像这样：

```
<Service Name>.<Namespace Name>.svc.cluster.local
```

通常，您只需要服务名称，DNS将自动解析为完整地址。 但是，如果需要访问另一个命名空间中的服务，则需使用服务名称加上命名空间名称。

例如，如果要连接到`test`命名空间中的`database`服务，可以使用以下地址：

```
database.test
```

如果要连接到`production`命名空间中的`database`服务，可以使用以下地址：

```
database.production
```

警告：如果您创建一个映射到“com”或“org”等TLD的命名空间，然后创建一个与网站名称相同的服务，例如“google”或“reddit”，Kubernetes将拦截“google.com“或”reddit.com“的请求并将其发送到您的服务。 这通常对于测试和代理非常有用，但也可以轻松破坏集群中的内容！

注意：如果确实要隔离命名空间，则应使用网络策略（Network Policies）来完成此操作。 更多信息请继续关注未来剧集！

## 命令空间粒度

一个常见问题是要创建多少个命名空间以及用于何种目的。 什么是可管理的块？ 创建太多的命名空间，它们会让你变得没有效率，但是创建太少，你会错过它们的好处。

我认为答案在于您的项目或公司处于什么阶段 - 从小团队到成熟企业，每个阶段都有自己的组织架构。 根据您的具体情况，您可以采用对应的命名空间策略。

### 小团队

在这种情况下，您是一个小团队的一员，该团队正在开发5-10个微服务，可以轻松地进行团队沟通。 在这种情况下，将所有生产服务放到“默认”命名空间是个不错的选择。 根据你的个人喜好，你可能希望有一个`production`和`development`的命名空间，但你很有可能是在本地机器上使用类似Minikube搭建的开发环境。

### 快速成长的团队

在这种情况下，您有一个快速发展的团队，正在开发10多个微服务。 您开始将团队分成多个子团队，每个团队都拥有自己的微服务。 虽然每个人都可能知道整个系统是如何工作的，但是与其他人协调每一个变化变得越来越困难。 尝试在本地计算机上启动整个堆栈每天都变得越来越复杂。

此时有必要使用多个集群或命名空间进行生产和开发。 每个团队可以选择拥有自己的命名空间，以便于管理。

### 大公司

在一家大公司，并不是每个人都了解其他人。 某个团队正致力于其他团队可能不了解的功能。 团队可能正在使用服契约(service contract)与其他微服务通信（例如，gRPC），也有可能使用服务网格(ServiceMesh)进行通信（例如，istio）。 试图在本地运行整个堆栈是不可能的。 强烈建议使用`Kubernetes-aware Continuous Delivery`系统（例如，Spinnaker）。

此时，每个团队肯定需要自己的命名空间。 每个团队甚至可以选择多个名称空间来运行其开发和生产环境。 设置`RBAC`和`ResourceQuotas`也是一个好主意。 多个集群开始显得很有意义，但可能不一定是必要的。

注意：我将在未来的剧集中深入研究gRPC，Istio，Spinnaker，RBAC和Resources！

### 企业

在这种规模下，有些群体甚至不知道其他群体的存在。 某个组也可能是外部公司，服务之间通过标准文档定义的API来通信。 每个小组都有多个拥有一定数量微服务的团队。 这时使用我上面提到的所有工具是必要的。 人们不应该手工部署服务，同时应该被锁定在他们不拥有的命名空间之外。

此时，拥有多个集群以减少配置不当的应用程序导致的爆炸半径，以及简化计费和资源管理可能是有意义的。

## 结论

命名空间可以帮助您组织Kubernetes资源，同时可以提高团队的开发速度。 请继续关注未来的Kubernetes最佳实践剧集，我将向您展示如何锁定命名空间中的资源并为您的群集引入更多安全性和隔离性！















