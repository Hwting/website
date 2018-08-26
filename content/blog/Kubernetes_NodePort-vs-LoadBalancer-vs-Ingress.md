---
title: "Kubernetes_NodePort vs LoadBalancer vs Ingress"
date: 2018-08-26T14:05:00+08:00
draft: true
banner: "https://cdn-images-1.medium.com/max/1600/1*KIVa4hUVZxg-8Ncabo8pdg.png"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "最近，有人问我NodePorts，LoadBalancers和Ingress之间有什么区别。 它们都是将外部流量引入群集的方式，但是分别以不同的方式完成。 让我们来具体看看它们是如何工作的，以及何时使用它们。"
tags: ["Kubernetes"]
categories: ["Kubernetes"]
---

# Kubernetes NodePort vs LoadBalancer vs Ingress

[原文](https://medium.com/google-cloud/kubernetes-nodeport-vs-loadbalancer-vs-ingress-when-should-i-use-what-922f010849e0)

最近，有人问我NodePorts，LoadBalancers和Ingress之间有什么区别。 它们都是将外部流量引入群集的方式，但是分别以不同的方式完成。 让我们来具体看看它们是如何工作的，以及何时使用它们。

注意：此处的所有内容均适用于Google Kubernetes Engine。 如果您在另一个云上运行，使用minikube或其他东西，这些将略有不同。 我也没有深入了解技术细节。 如果您有兴趣了解更多信息，官方文档是一个很好的资源！

## ClusterIP

ClusterIP服务是默认的Kubernetes服务。 它为您提供集群内的其他应用程序可以访问的服务。 没有外部访问权限。

ClusterIP服务的YAML如下所示：

```yaml
apiVersion: v1
kind: Service
metadata:  
  name: my-internal-service
spec:
  selector:    
    app: my-app
  type: ClusterIP
  ports:  
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
```

如果您无法从互联网访问ClusterIP服务，我为什么要谈论它呢？ 事实证明，您可以使用Kubernetes代理访问它！

![1*I4j4xaaxsuchdvO66V3lAg.png](https://cdn-images-1.medium.com/max/1600/1*I4j4xaaxsuchdvO66V3lAg.png)

启动Kubernetes代理：

```shell
$ kubectl proxy --port=8080
```

现在，您可以使用Kubernetes API以访问此服务：

```
http://localhost:8080/api/v1/proxy/namespaces/<NAMESPACE>/services/<SERVICE-NAME>:<PORT-NAME>/
```

因此，要访问我们上面定义的服务，您可以使用以下地址：

```
http://localhost:8080/api/v1/proxy/namespaces/default/services/my-internal-service:http/
```

### 什么时候使用Kubernetes Proxy访问服务?

在某些情况下，您可以使用Kubernetes代理来访问您的服务。

1. 调试您的服务或出于某种原因直接从您的笔记本电脑连接它们
2. 允许内部流量，显示内部dashboard等

因为此方法要求您作为经过身份验证的用户运行kubectl，所以不应使用此方法将服务公开给Internet或将其用于生产服务。

## NodePort

NodePort服务是将外部流量直接发送给您的服务的最原始方式。 顾名思义，NodePort在所有节点（VM）上打开一个特定端口，并且发送到该端口的任何流量都将转发到该服务。

![1*CdyUtG-8CfGu2oFC5s0KwA.png](https://cdn-images-1.medium.com/max/1600/1*CdyUtG-8CfGu2oFC5s0KwA.png)

注意：上图并不是技术上最精确的图示，但我认为它说明了NodePort的工作原理。

NodePort服务的YAML如下所示：

```yaml
apiVersion: v1
kind: Service
metadata:  
  name: my-nodeport-service
spec:
  selector:    
    app: my-app
  type: NodePort
  ports:  
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30036
    protocol: TCP
```

基本上，NodePort服务与普通的“ClusterIP”服务有两点不同。 首先，类型是“NodePort”。还有一个名为nodePort的附加端口，用于指定在节点上打开的端口。 如果您不指定此端口，它将选择一个随机端口。 大多数时候你应该让Kubernetes选择端口; 正如thockin所说，对于你可以使用的端口有很多必要的说明。

### 什么时候使用这种方法？

这种方法有许多缺点：

1. 每个端口只能有一个服务
2. 您只能使用端口30000-32767
3. 如果您的Node/VM的IP地址发生变化，您需要处理好

出于这些原因，我不建议在生产中使用此方法直接公开您的服务。 如果您运行的服务不必始终可用，或者您的成本非常敏感，则此方法适合您。 比如演示应用程序或临时的东西。

## LoadBalancer

LoadBalancer服务是将服务公开给Internet的标准方法。 在GKE上，这将启动`Network Load Balancer`，它将为您提供单个IP地址，以便将所有流量转发到您的服务。

![1*P-10bQg_1VheU9DRlvHBTQ.png](https://cdn-images-1.medium.com/max/1600/1*P-10bQg_1VheU9DRlvHBTQ.png)

### 什么时候使用？

如果要直接公开服务，这是默认方法。 您指定的端口上的所有流量都将转发到该服务。 没有过滤，没有路由等。这意味着您可以向其发送几乎任何类型的流量，如HTTP，TCP，UDP，Websockets，gRPC等等。

最大的缺点是您使用LoadBalancer公开的每个服务都将获得自己的IP地址，并且您必须为每个公开的服务支付LoadBalancer，这可能会变得昂贵！

## Ingress

与上述所有示例不同，Ingress实际上不是一种服务。 相反，它位于多个服务的前面，充当集群中的“智能路由器”或入口点。

您可以使用Ingress执行许多不同的操作，并且有许多类型的Ingress控制器，都具有不同的功能。

默认的GKE ingress controller将为您启动HTTP(S)负载均衡器。 这将允许您执行基于路径和子域的路由到后端服务。 例如，您可以将`foo.yourdomain.com`上的所有内容发送到foo服务，并将`youdomain.com/bar/`下的所有内容发送到bar服务。

![1*KIVa4hUVZxg-8Ncabo8pdg.png](https://cdn-images-1.medium.com/max/1600/1*KIVa4hUVZxg-8Ncabo8pdg.png)

使用L7 HTTP负载均衡器的GKE上的Ingress对象的YAML可能如下所示：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
spec:
  backend:
    serviceName: other
    servicePort: 8080
  rules:
  - host: foo.mydomain.com
    http:
      paths:
      - backend:
          serviceName: foo
          servicePort: 8080
  - host: mydomain.com
    http:
      paths:
      - path: /bar/*
        backend:
          serviceName: bar
          servicePort: 8080
```

### 什么时候使用？

Ingress可能是暴露您的服务的最有效方式，但也可能是最复杂的。 有许多类型的Ingress控制器，来自Google Cloud Load Balancer，Nginx，Contour，Istio等。 还有Ingress控制器的插件，如cert-manager，可以为您的服务自动配置SSL证书。

如果要在同一IP地址下公开多个服务，Ingress是最有用的，并且这些服务都使用相同的L7协议（通常是HTTP）。 如果您使用GCP集成，您只需支付一个负载均衡器，并且因为Ingress是“智能”的，您可以获得大量开箱即用的功能（如SSL，身份验证，路由等）。












