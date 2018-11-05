---
title: "监控多个Kubernetes集群"
date: 2018-11-05T10:27:00+08:00
draft: false
banner: "https://cdn-images-1.medium.com/max/1600/1*qi72SThV3Cai-pJoVl6lxw.png"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "在 THG，我们为多个团队管理 Kubernetes 集群。 为了有效地监控这些集群，我们在每个数据中心使用单个 Prometheus 实例。"
tags: ["Kubernetes", "Prometheus"]
categories: ["Kubernetes"]
---

# 监控多个 Kubernetes 集群

[原文](https://medium.com/thg-tech-blog/monitoring-multiple-kubernetes-clusters-88cf34442fa3)

在 THG，我们为多个团队管理 Kubernetes 集群。 为了有效地监控这些集群，我们在每个数据中心使用单个 Prometheus 实例。

Prometheus 是一个开源监控工具，Kubernetes 支持其开箱即用，以 Prometheus 格式公开端点上的集群运行状况和操作指标。 Prometheus 还支持使用 Kubernetes REST API 作为源来发现在集群内运行的其他标准度量指标。

## Cluster 身份认证

我们需要做的第一件事是创建一个 service account，Prometheus 实例将使用该帐户对集群进行身份验证。

要配置 service account 可以访问的内容，您需要设置 clusterrole 和 clusterrolebinding。 以下是 clusterrole 为 Prometheus 提供对我们感兴趣的每个资源的读取访问权限。

[prometheus-querier.yaml · GitHub](https://gist.github.com/ConorNevinTHG/1818ac894dc41eeee10f1835c85e81d2#file-prometheus-querier-yaml)

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus-querier
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/proxy
      - services
      - services/proxy
      - endpoints
      - pods
      - pods/proxy
    verbs: ["get", "list", "watch"]
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]
```

一旦我们创建了 clusterrole，我们就需要通过运行

```shell
kubectl create clusterrolebinding prometheus-querier -clusterrole=prometheus-querier -serviceaccount=kube-system:prometheus
```

将它绑定到 service account。

现在我们已经配置了我们的集群，接着我们需要配置 Prometheus 实例。

## Prometheus 配置

Prometheus 有一个用于 Kubernetes 的简单示例配置; 但是，它是从集群内部运行，并假设默认值在集群外部不起作用。

在集群内部，这就是发现所有节点所需的全部配置。

[in-cluster-node-config.yaml · GitHub](https://gist.github.com/ConorNevinTHG/704b75a717e6f231a60a48b6c6b7c971#file-in-cluster-node-config-yaml)

```yaml
kubernetes_sd_configs:
  - role: node
```

为了使起工作在集群之外，我们需要指向与我们之前创建的 service account 相关联的令牌以及集群的 CA（证书颁发机构）和 Kubernetes REST API 的地址。

[out-of-cluster-node-config.yaml · GitHub](https://gist.github.com/ConorNevinTHG/2a30750c50a352dd7bc5643ef7be128b#file-out-of-cluster-node-config-yaml)

```yaml
kubernetes_sd_configs:
  - api_server: <api server address>
    role: node
    bearer_token_file: <path to the token file>
    tls_config:
      ca_file: <path to the ca file>
```

这将允许 Prometheus 实例去构建它需要抓取的目标列表，但我们还需要将 token 和 CA 文件添加到配置中，以便能够成功地抓取指标。

现在我们可以向集群发出请求了，但是我们需要进行一些 relabel，以便 Prometheus 实例能够构建正确的外部 URL 以到达目标。

此 relabel 配置将每个标签作为 Prometheus 标签加载到相应节点，将目标地址重写为 API 服务器的地址，并更改度量标准路径，这样就使用 Kubernetes API 上的代理端点。

[node-relabel-config.yaml · GitHub](https://gist.github.com/ConorNevinTHG/758cbed9941655314a3749fe12d8edbc#file-node-relabel-config-yaml)

```yaml
relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
  - target_label: __address__
    replacement: <api server address>
  - source_labels: [__meta_kubernetes_node_name]
    regex: (.+)
    target_label: __metrics_path__
    replacement: /api/v1/nodes/${1}/proxy/metrics
```

同时，我们还为每个标识集群的度量标准添加了新的静态标签，因此我们可以轻松区分属于不同集群的度量标准。 最终配置如下所示:

[full-out-of-cluster-node-config.yaml · GitHub](https://gist.github.com/ConorNevinTHG/55ede402ca9a8a1c4a6661b73babe39c#file-full-out-of-cluster-node-config-yaml)

```yaml
bearer_token_file: <path to the token file>
tls_config:
  ca_file: <path to the ca file>
scheme: https
kubernetes_sd_configs:
  - api_server: <api server address>
    role: node
    bearer_token_file: <path to the token file>
    tls_config:
      ca_file: <path to the ca file>
relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
  - target_label: __address__
    replacement: <api server address>
  - source_labels: [__meta_kubernetes_node_name]
    regex: (.+)
    target_label: __metrics_path__
    replacement: /api/v1/nodes/${1}/proxy/metrics
  - source_labels: []
    action: replace
    target_label: kubernetes_cluster
    replacement: <cluster name or id>
```

有了这个配置，我们仍然需要改变配置来抓取其他目标，node-cadvisor，pods 和服务。 为了抓取 cadvisor 指标，我们需要做的就是复制上面的配置并将指标路径更改为`/api/v1/nodes/${1}/proxy/metrics/cadvisor`。 对于 pod 和服务，我们需要使用以下 relabel 配置:

[pod-relabel-config.yaml · GitHub](https://gist.github.com/ConorNevinTHG/0159f6e02b1fb16f546fd0eeefb7cfe2#file-pod-relabel-config-yaml)

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true
  - target_label: __address__
    replacement: <api server address>
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scheme]
    regex: ^$
    replacement: http
    target_label: __meta_kubernetes_pod_annotation_prometheus_io_scheme
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
    regex: (.+)
    replacement: ${1}
    target_label: __metrics_path__
  - source_labels:
      - __meta_kubernetes_namespace
      - __meta_kubernetes_pod_annotation_prometheus_io_scheme
      - __meta_kubernetes_pod_name
      - __meta_kubernetes_pod_annotation_prometheus_io_port
      - __metrics_path__
    regex: (.+);(.+);(.+);(.+);(.+)
    action: replace
    target_label: __metrics_path__
    replacement: /api/v1/namespaces/${1}/pods/${2}:${3}:${4}/proxy${5}
  - action: labelmap
    regex: __meta_kubernetes_pod_label_(.+)
```

此配置将过滤所有正在运行的 pod 的列表，并只抓取带有注解`prometheus.io/scrape=true`的目标，然后使用其他注解构建可抓取地址，以允许我们配置度量端点的端口和路径。

[service-relabel-config.yaml · GitHub](https://gist.github.com/ConorNevinTHG/2d335977daf6e689d17ed581df83ef6f#file-service-relabel-config-yaml)

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
    action: keep
    regex: true
  - target_label: __address__
    replacement: <api server address>
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
    regex: ^$
    replacement: http
    target_label: __meta_kubernetes_service_annotation_prometheus_io_scheme
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
    regex: (.+)
    replacement: ${1}
    target_label: __metrics_path__
  - source_labels:
      - __meta_kubernetes_namespace
      - __meta_kubernetes_service_annotation_prometheus_io_scheme
      - __meta_kubernetes_service_name
      - __meta_kubernetes_service_annotation_prometheus_io_port
      - __metrics_path__
    regex: (.+);(.+);(.+);(.+);(.+)
    action: replace
    target_label: __metrics_path__
    replacement: /api/v1/namespaces/${1}/services/${2}:${3}:${4}/proxy${5}
  - action: labelmap
    regex: __meta_kubernetes_service_label_(.+)
```

服务的配置几乎没有改变，唯一的差别是使用服务注解构造略有不同的地址。

## 配置 Generation/Management

现在我们已经完成了所有这些工作，我们需要一种为每个我们想要抓取的集群自动生成此配置的方法，因为手动编写这个配置需要很长时间。 Prometheus 不支持从目录加载配置。 相反，它要求它全部存在于单个文件中，因此我们使用 Ansible 为我们生成文件。

Ansible 有一个模块，允许将多个文件组装成一个名为“assemble”的更大的单个文件，这将使得支持多种不同的 scrape 类型变得更加容易。 简而言之，它还支持验证步骤，我们可以在覆盖旧的验证步骤之前验证我们的配置是否正确。 通过将上面的配置重写为模板，我们可以将每个集群的配置输出到一个目录中，然后将它们组合成一个文件。

[ansible-assemble-config.yaml · GitHub](https://gist.github.com/ConorNevinTHG/e9e096c7b998298f57a9d82b9783c6c1#file-ansible-assemble-config-yaml)

```yaml
- name: Assemble prometheus config
  assemble:
    src: "{{ prometheus_config_path }}/fragments"
    dest: "{{ prometheus_config_path }}/prometheus.yaml"
    validate: "{{ prometheus_bin_path }}/promtool_{{ prometheus_version }} check config %s"
    backup: yes
```

我们现在有一个可以正常工作的 Prometheus 实例，您现在应该看到群集中的目标列表被自动抓取。

作为集群设置的一部分，我们在集群中安装了两个组件，为我们提供了一些额外的指标：

- Kube State Metrics，展示有关集群内各种资源的内部状态的指标
- Node Exporter, 展示集群中每个主机的基础级别度量指标

现在这一切都已到位，每次我们启动新集群时，我们需要做的就是重新生成我们的 Prometheus 配置，并且我们会自动从新集群中抓取所有指标！

![1*qi72SThV3Cai-pJoVl6lxw.png](https://cdn-images-1.medium.com/max/1600/1*qi72SThV3Cai-pJoVl6lxw.png)
