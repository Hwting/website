---
title: "Kubernetes101"
date: 2018-07-31T12:07:48+08:00
draft: false
banner: "https://www.stavros.io/posts/kubernetes-101/sail.jpg"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "Kubernetes 101"
tags: ["Kubernetes"]
categories: ["Kubernetes"]
---

# Kubernetes 101

[原文链接](https://www.stavros.io/posts/kubernetes-101/)

几个星期前，我的工作任务变得很有趣：部署`Kubernetes`集群并编写相关工具，以便开发人员可以在他们正在处理的分支中部署代码，并测试其更改。

在那之前，我一直想学习`Kubernetes`，因为它听起来很有意思（如果你是希腊人，你会觉得这个名字很有问题），但我从来没有机会，因为我没有任何东西需要运行在集群中。所以，这次我抓住机会，开始查资料，但所有的资料（包括官方教程）似乎太冗长，结构也不合理，所以我有点沮丧。

无论如何，经过几天的研究，事情终于有点眉目了，我开始疯狂在不同的机器上部署，迅速让我的AWS账单暴增了数千美元，这才像一个2018年的有自尊的后端开发人员。因为我的简历现在说自己是个“Kubernetes专家”，一个想法立刻诞生了：为什么不把我对这个系统的宽泛理解以及我已经耗费了几个小时的研究所收集的知识让更多人看到？虽然我无法说服自己不应该再写另一篇漫无目的的文章，但是我很快就明白了：

这就是那篇文章。

我在现有文章中遇到的主要问题是，在深入研究具体细节之前，我找不到的任何内容总结了这些组件是什么以及它们如何组合起来的高级概述。 而这种高屋建瓴的呈现方式是我学习最好的方式。我是以这种方式来写的，希望它也适合你。如果你知道任何描述了`Kubernetes`如何工作，而且让人容易理解的专家级的文章/教程，请不要告诉我，因为你在我需要你的时候你在哪里，现在我写了我的文章而你却没有及早把它拿出来。

另外请记住，我实际上只学习了`Kubernetes`一个星期左右，所以学得不会非常深入，有些可能是不准确的，尽管希望没有什么错误，这里的信息应该足够让你达到运行简单集群的程度。

话虽如此，最后我发现`Kubernetes`中的概念还是非常简单的，虽然我确信有很多东西我还不知道。但是，我知道的事情就足以建立一个集群并让我们的应用在其上运行，而且我很确定它们足以让大多数人知道如何开始。

## 基本概念

![captain.jpg](https://www.stavros.io/posts/kubernetes-101/captain.jpg)

我们需要做的第一件事是详细介绍`Kubernetes`的各个部分：

* 控制平面(Control plane)：顾名思义，这是控制其他一切的部分，这也是我一无所知的部分，因为我们只是向亚马逊付费，让亚马逊帮我们处理这部分。我的理解是，这是最好的决定，除非你是谷歌，否则你应该付费给一些公司，让他们为你管理。

* 节点(Nodes)：节点本质上就是一台服务器，就像您付费的物理机worker一样。 这是所有代码部署的地方，将裸服务器变成节点的方法是在其上安装`Docker`，`kubelet`，`kube-proxy`和其他一些东西。本文假设您的群集中已有一些worker。

* 容器集(Pods)：pod是容器集合。 这是您的代码所在的位置，通常每个容器都有一个pod，尽管您可能希望将一些密切相关的服务放在同一个pod中。 pod在单个节点上运行（但是一个节点可以运行许多pod），这意味着pod中的所有容器将具有相同的IP地址，并且它们可以通过连接到localhost上的彼此端口来相互通信。Pod在部署后无法更新，只能删除或替换它们。

* 部署(Deployments)： `Deployment`是您将pod实际部署到群集的方式。 您可以在没有`Deployment`的情况下运行pod，但如果没有`Deployment`，则无法轻松指定所需的副本数量，在失败时自动重新部署pod，回滚到早期状态等。`Deployment`使代码生命周期管理变得更容易，并且您可以使用它来使Docker镜像在Kubernetes上运行。

* 服务(Service)：服务允许您从一个pod打开端口到其他pod，并指定一个pod的DNS名称，以便能够查找并连接到群集中的其他pod。

* 入口(Ingress)：`Ingresses`是你如何告诉你的Ingress控制器（通常是像`Traefik`这样的`web server`）向外界暴露什么，以及在哪个路径或主机名上。 入口将`https://some-hostname.your-cluster.your-company.com`映射到将实际应答该请求的pod。本教程也假设您已经配置了入口，虽然设置`Traefik`来做到这一点不应该非常困难（在用他们的教程时请使用Deployment方法）。

所有这些都可以使用命令行的`kubectl`创建，或者更安全地通过YAML文件创建，该文件将包含您要部署的内容的定义和详细信息（然后执行`kubectl apply -f <yaml file>`。

概括地讲，您把容器放入`pods`中，这些`pods`将由`deployment`创建和部署，其网络将由`service`处理，并添加`ingress`以便外部世界可以访问您的服务器。

让我们逐个介绍这些部分，看看它们的YAML配置是什么样的。

## The Pod

![pods.jpg](https://www.stavros.io/posts/kubernetes-101/pods.jpg)

让我们看一下将在容器中运行`redis`镜像的pod的YAML配置。 请记住，pod并不是持久性的，所以你几乎不会直接使用它。 相反，您将使用`deployment`间接部署`pod`，我们将在下面介绍。

以下配置示例仅供您进行修改。 你只需要看看它，然后继续阅读，不要停下来惊叹它的美丽。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod-name
spec:
  containers:
  - name: my-redis
    image: redis
    ports:
    - containerPort: 6379
```

正如您所看到的，它非常简单，您添加了一堆Kubernetes特定的东西，每个都只是复制粘贴，然后您声明此配置是为Pod，给它一个名称，指定在其中运行的容器和他们监听的端口，请删除整个文件吧，你已经准备好了！

Kubernetes官方文档中提供了更多关于[Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/)的信息。

## The deployment

以下是您实际运行上述pod的方式，即使用deployment。 请记住，您根本不需要关注上面的pod配置，我们将在deployment里重新定义它。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment-name
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: my-redis-pod
    spec:
      containers:
      - name: my-redis
        image: redis
        ports:
        - containerPort: 6379
```

您会注意到这主要是上面的Pod配置，但有一些额外的配置，如副本(replica)等。这些定义了deployment的名称以及我们要部署的副本数量。 更改副本数量，将会部署更多`template`部分中指定的pod。

Kubernetes官方文档中提供了更多关于[Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)的信息。

## The service

![sail.jpg](https://www.stavros.io/posts/kubernetes-101/sail.jpg)

现在我们已经部署了一个pod，我们需要将其端口暴露给集群的其余部分。 部署中的`containerPort`指令暴露了Docker端口，但实际上并不转发主机上的端口，因此多个pod（不是同一pod中的容器）可以使用相同的端口而不会发生冲突。

要将上面的端口实际暴露给集群上运行的其他pod，我们需要为它创建一个Service。 这将创建转发端口所需的规则，并为我们提供DNS条目，我们可以使用该条目来解析该Pod的IP。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  ports:
  - port: 6379
    name: redis-port
    targetPort: 6379
    protocol: TCP
  selector:
    app: my-redis-pod
```

这会将redis端口暴露给集群中的其他pod，可以通过`my-service：6379`连接它。

要部署你的应用中更多部分，只需将另一个deployment和关联的Service添加到群集即可。 您可以使用与上面的redis完全相同的方式部署主应用程序服务。

## The ingress

最后，我们可以使用Ingress将我们的服务暴露给互联网。 这里是使用Traefik的一个例子，虽然您可能实际上并不想将Redis暴露给外面的世界，但同样的方法适用于您自己的应用程序。

```yaml
apiVersion: extensions/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: redis.yourdomain.com
    http:
      paths:
      - backend:
          serviceName: my-service
          servicePort: 6379
```

这一节配置是告诉Traefik你希望所有名为`redis.yourdomain.com`的主机上的流量都转发到我的服务端口6379.据我所知，这只是针对Traefik的配置。 在应用配置后，pod将通过`redis.yourdomain.com`上的Traefik暴露到互联网。

## 结语

我希望这篇文章对初学者有用。这篇文章很简短，因为Kubernetes的基础很短，但我们设法涵盖了如何以最小的麻烦来运行服务。

如果帖子中有任何不准确或错误（或者如果您有任何要添加的内容），请通过[Twitter](https://twitter.com/intent/user?screen_name=Stavros)或 [tooting](https://mastodon.host/@stavros) 告诉我。

现在你应该懂Kubernetes是什么了！




