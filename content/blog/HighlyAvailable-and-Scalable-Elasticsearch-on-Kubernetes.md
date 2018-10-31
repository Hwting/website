---
title: "HighlyAvailable and Scalable Elasticsearch on Kubernetes"
date: 2018-10-31T14:41:34+08:00
draft: false
banner: "https://cdn-images-1.medium.com/max/1600/1*HXdNCVY9Mhx_gWyYIBjKow.jpeg"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "在上一篇文章中，我们通过扩展MongoDB副本集来了解有StatefulSets。 在这篇文章中，我们将与ES-HQ和Kibana一起使用HA Elasticsearch集群（具有不同的Master，Data和Client节点）。"
tags: ["Kubernetes", "Elasticsearch"]
categories: ["Kubernetes"]
---

# Kubernetes上部署高可用和可扩展的Elasticsearch

[原文](https://medium.com/devopslinks/https-medium-com-thakur-vaibhav23-ha-es-k8s-7e655c1b7b61)

在上一篇文章中，我们通过扩展MongoDB副本集来了解有StatefulSets。 在这篇文章中，我们将与ES-HQ和Kibana一起使用HA Elasticsearch集群（具有不同的Master，Data和Client节点）。

### 先决条件

1. Elasticsearch的基本知识，其Node类型及角色
2. 运行至少有3个节点的Kubernetes集群（至少4Cores 4GB）
3. Kibana的相关知识

### 部署架构图

![1*TG61ojlzv2v-akGqwws2Ew.jpeg](https://cdn-images-1.medium.com/max/1600/1*TG61ojlzv2v-akGqwws2Ew.jpeg)

- Elasticsearch Data Node的Pod被部署为具有Headless Service的StatefulSets，以提供稳定的网络ID
- Elasticsearch Master Node的Pod被部署为具有Headless Service的副本集，这将有助于自动发现
- Elasticsearch Client Node的Pod部署为具有内部服务的副本集，允许访问R/W请求的Data Node
- Kibana和ElasticHQ Pod被部署为副本集，其服务可在Kubernetes集群外部访问，但仍在您的子网内部（除非另有要求，否则不公开）
- 为Client Node部署HPA（Horizonal Pod Auto-scaler）以在高负载下实现自动伸缩

### 要记住的重要事项：

1. 设置ES_JAVA_OPT环境变量
2. 设置CLUSTER_NAME环境变量
3. 为Master Node的部署设置NUMBER_OF_MASTERS环境变量（防止脑裂问题）。如果有3个Masters，我们必须设置为2。
4. 在类似的pod中设置正确的Pod-AntiAffinity策略，以便在工作节点发生故障时确保HA。

让我们直接将这些服务部署到我们的GKE集群。

### Master节点部署

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: elasticsearch
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: es-master
  namespace: elasticsearch
  labels:
    component: elasticsearch
    role: master
spec:
  replicas: 3
  template:
    metadata:
      labels:
        component: elasticsearch
        role: master
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: role
                  operator: In
                  values:
                  - master
              topologyKey: kubernetes.io/hostname
      initContainers:
      - name: init-sysctl
        image: busybox:1.27.2
        command:
        - sysctl
        - -w
        - vm.max_map_count=262144
        securityContext:
          privileged: true
      containers:
      - name: es-master
        image: quay.io/pires/docker-elasticsearch-kubernetes:6.2.4
        env:
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: CLUSTER_NAME
          value: my-es
        - name: NUMBER_OF_MASTERS
          value: "2"
        - name: NODE_MASTER
          value: "true"
        - name: NODE_INGEST
          value: "false"
        - name: NODE_DATA
          value: "false"
        - name: HTTP_ENABLE
          value: "false"
        - name: ES_JAVA_OPTS
          value: -Xms256m -Xmx256m
        - name: PROCESSORS
          valueFrom:
            resourceFieldRef:
              resource: limits.cpu
        resources:
          limits:
            cpu: 2
        ports:
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: storage
          mountPath: /data
      volumes:
          - emptyDir:
              medium: ""
            name: "storage"
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-discovery
  namespace: elasticsearch
  labels:
    component: elasticsearch
    role: master
spec:
  selector:
    component: elasticsearch
    role: master
  ports:
  - name: transport
    port: 9300
    protocol: TCP
  clusterIP: None
```


```shell
root$ kubectl apply -f es-master.yml
root$ kubectl -n elasticsearch get all
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/es-master   3         3         3            3           32s
NAME                      DESIRED   CURRENT   READY     AGE
rs/es-master-594b58b86c   3         3         3         31s
NAME                            READY     STATUS    RESTARTS   AGE
po/es-master-594b58b86c-9jkj2   1/1       Running   0          31s
po/es-master-594b58b86c-bj7g7   1/1       Running   0          31s
po/es-master-594b58b86c-lfpps   1/1       Running   0          31s
NAME                          TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
svc/elasticsearch-discovery   ClusterIP   None         <none>        9300/TCP   31s
```

有趣的是，可以从任何主节点pod的日志来见证它们之间的master选举，然后何时添加新的data和client节点。

```shell
root$ kubectl -n elasticsearch logs -f po/es-master-594b58b86c-9jkj2 | grep ClusterApplierService
[2018-10-21T07:41:54,958][INFO ][o.e.c.s.ClusterApplierService] [es-master-594b58b86c-9jkj2] detected_master {es-master-594b58b86c-bj7g7}{1aFT97hQQ7yiaBc2CYShBA}{Q3QzlaG3QGazOwtUl7N75Q}{10.9.126.87}{10.9.126.87:9300}, added {{es-master-594b58b86c-lfpps}{wZQmXr5fSfWisCpOHBhaMg}{50jGPeKLSpO9RU_HhnVJCA}{10.9.124.81}{10.9.124.81:9300},{es-master-594b58b86c-bj7g7}{1aFT97hQQ7yiaBc2CYShBA}{Q3QzlaG3QGazOwtUl7N75Q}{10.9.126.87}{10.9.126.87:9300},}, reason: apply cluster state (from master [master {es-master-594b58b86c-bj7g7}{1aFT97hQQ7yiaBc2CYShBA}{Q3QzlaG3QGazOwtUl7N75Q}{10.9.126.87}{10.9.126.87:9300} committed version [3]])
```

可以看出，名为es-master-594b58b86c-bj7g7的es-master pod被选为master节点，其他2个pod被添加到这个集群。

名为elasticsearch-discovery的Headless Service默认设置为docker镜像中的env变量，用于在节点之间进行发现。 当然这是可以被改写的。

同样，我们可以部署Data和Client节点。 配置如下：

### Data节点部署：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: elasticsearch
---
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  fsType: xfs
allowVolumeExpansion: true
---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: es-data
  namespace: elasticsearch
  labels:
    component: elasticsearch
    role: data
spec:
  serviceName: elasticsearch-data
  replicas: 3
  template:
    metadata:
      labels:
        component: elasticsearch
        role: data
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: role
                  operator: In
                  values:
                  - data
              topologyKey: kubernetes.io/hostname
      initContainers:
      - name: init-sysctl
        image: busybox:1.27.2
        command:
        - sysctl
        - -w
        - vm.max_map_count=262144
        securityContext:
          privileged: true
      containers:
      - name: es-data
        image: quay.io/pires/docker-elasticsearch-kubernetes:6.2.4
        env:
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: CLUSTER_NAME
          value: my-es
        - name: NODE_MASTER
          value: "false"
        - name: NODE_INGEST
          value: "false"
        - name: HTTP_ENABLE
          value: "false"
        - name: ES_JAVA_OPTS
          value: -Xms256m -Xmx256m
        - name: PROCESSORS
          valueFrom:
            resourceFieldRef:
              resource: limits.cpu
        resources:
          limits:
            cpu: 2
        ports:
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: storage
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: storage
      annotations:
        volume.beta.kubernetes.io/storage-class: "fast"
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: fast
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-data
  namespace: elasticsearch
  labels:
    component: elasticsearch
    role: data
spec:
  ports:
  - port: 9300
    name: transport
  clusterIP: None
  selector:
    component: elasticsearch
    role: data

```

Headless Service为Data节点提供稳定的网络ID，有助于它们之间的数据传输。

在将持久卷附加到pod之前格式化它是很重要的。 这可以通过在创建storage class时指定卷类型来完成。 我们还可以设置标志以允许动态扩展。 [这里](https://kubernetes.io/blog/2018/07/12/resizing-persistent-volumes-using-kubernetes/)可以阅读更多内容。

```yaml
...
parameters:  
  type: pd-ssd  
  fsType: xfs
allowVolumeExpansion: true
...
```

### Client节点部署

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: elasticsearch
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: es-client
  namespace: elasticsearch
  labels:
    component: elasticsearch
    role: client
spec:
  replicas: 2
  template:
    metadata:
      labels:
        component: elasticsearch
        role: client
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: role
                  operator: In
                  values:
                  - client
              topologyKey: kubernetes.io/hostname
      initContainers:
      - name: init-sysctl
        image: busybox:1.27.2
        command:
        - sysctl
        - -w
        - vm.max_map_count=262144
        securityContext:
          privileged: true
      containers:
      - name: es-client
        image: quay.io/pires/docker-elasticsearch-kubernetes:6.2.4
        env:
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: CLUSTER_NAME
          value: my-es
        - name: NODE_MASTER
          value: "false"
        - name: NODE_DATA
          value: "false"
        - name: HTTP_ENABLE
          value: "true"
        - name: ES_JAVA_OPTS
          value: -Xms256m -Xmx256m
        - name: NETWORK_HOST
          value: _site_,_lo_
        - name: PROCESSORS
          valueFrom:
            resourceFieldRef:
              resource: limits.cpu
        resources:
          limits:
            cpu: 1
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: storage
          mountPath: /data
      volumes:
          - emptyDir:
              medium: ""
            name: storage
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: elasticsearch
  annotations: 
    cloud.google.com/load-balancer-type: Internal
  labels:
    component: elasticsearch
    role: client
spec:
  selector:
    component: elasticsearch
    role: client
  ports:
  - name: http
    port: 9200
  type: LoadBalancer
```

此处部署的服务是从Kubernetes集群外部访问ES群集，但仍在我们的子网内部。 注释掉`cloud.google.com/load-balancer-type：Internal`可确保这一点。

但是，如果我们的ES集群中的应用程序部署在集群中，则可以通过 http://elasticsearch.elasticsearch:9200 来访问ElasticSearch服务。

创建这两个deployments后，新创建的client和data节点将自动添加到集群中。（观察master pod的日志）

```shell
root$ kubectl apply -f es-data.yml
root$ kubectl -n elasticsearch get pods -l role=data
NAME        READY     STATUS    RESTARTS   AGE
es-data-0   1/1       Running   0          48s
es-data-1   1/1       Running   0          28s
--------------------------------------------------------------------
root$ kubectl apply -f es-client.yml 
root$ kubectl -n elasticsearch get pods -l role=client
NAME                         READY     STATUS    RESTARTS   AGE
es-client-69b84b46d8-kr7j4   1/1       Running   0          47s
es-client-69b84b46d8-v5pj2   1/1       Running   0          47s
--------------------------------------------------------------------
root$ kubectl -n elasticsearch get all
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/es-client   2         2         2            2           1m
deploy/es-master   3         3         3            3           9m
NAME                      DESIRED   CURRENT   READY     AGE
rs/es-client-69b84b46d8   2         2         2         1m
rs/es-master-594b58b86c   3         3         3         9m
NAME                   DESIRED   CURRENT   AGE
statefulsets/es-data   2         2         3m
NAME                            READY     STATUS    RESTARTS   AGE
po/es-client-69b84b46d8-kr7j4   1/1       Running   0          1m
po/es-client-69b84b46d8-v5pj2   1/1       Running   0          1m
po/es-data-0                    1/1       Running   0          3m
po/es-data-1                    1/1       Running   0          3m
po/es-master-594b58b86c-9jkj2   1/1       Running   0          9m
po/es-master-594b58b86c-bj7g7   1/1       Running   0          9m
po/es-master-594b58b86c-lfpps   1/1       Running   0          9m
NAME                          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
svc/elasticsearch             LoadBalancer   10.9.121.160 10.9.120.8     9200:32310/TCP   1m
svc/elasticsearch-data        ClusterIP   None           <none>        9300/TCP         3m
svc/elasticsearch-discovery   ClusterIP   None           <none>        9300/TCP         9m
--------------------------------------------------------------------
#Check logs of es-master leader pod
root$ kubectl -n elasticsearch logs po/es-master-594b58b86c-bj7g7 | grep ClusterApplierService
[2018-10-21T07:41:53,731][INFO ][o.e.c.s.ClusterApplierService] [es-master-594b58b86c-bj7g7] new_master {es-master-594b58b86c-bj7g7}{1aFT97hQQ7yiaBc2CYShBA}{Q3QzlaG3QGazOwtUl7N75Q}{10.9.126.87}{10.9.126.87:9300}, added {{es-master-594b58b86c-lfpps}{wZQmXr5fSfWisCpOHBhaMg}{50jGPeKLSpO9RU_HhnVJCA}{10.9.124.81}{10.9.124.81:9300},}, reason: apply cluster state (from master [master {es-master-594b58b86c-bj7g7}{1aFT97hQQ7yiaBc2CYShBA}{Q3QzlaG3QGazOwtUl7N75Q}{10.9.126.87}{10.9.126.87:9300} committed version [1] source [zen-disco-elected-as-master ([1] nodes joined)[{es-master-594b58b86c-lfpps}{wZQmXr5fSfWisCpOHBhaMg}{50jGPeKLSpO9RU_HhnVJCA}{10.9.124.81}{10.9.124.81:9300}]]])
[2018-10-21T07:41:55,162][INFO ][o.e.c.s.ClusterApplierService] [es-master-594b58b86c-bj7g7] added {{es-master-594b58b86c-9jkj2}{x9Prp1VbTq6_kALQVNwIWg}{7NHUSVpuS0mFDTXzAeKRcg}{10.9.125.81}{10.9.125.81:9300},}, reason: apply cluster state (from master [master {es-master-594b58b86c-bj7g7}{1aFT97hQQ7yiaBc2CYShBA}{Q3QzlaG3QGazOwtUl7N75Q}{10.9.126.87}{10.9.126.87:9300} committed version [3] source [zen-disco-node-join[{es-master-594b58b86c-9jkj2}{x9Prp1VbTq6_kALQVNwIWg}{7NHUSVpuS0mFDTXzAeKRcg}{10.9.125.81}{10.9.125.81:9300}]]])
[2018-10-21T07:48:02,485][INFO ][o.e.c.s.ClusterApplierService] [es-master-594b58b86c-bj7g7] added {{es-data-0}{SAOhUiLiRkazskZ_TC6EBQ}{qirmfVJBTjSBQtHZnz-QZw}{10.9.126.88}{10.9.126.88:9300},}, reason: apply cluster state (from master [master {es-master-594b58b86c-bj7g7}{1aFT97hQQ7yiaBc2CYShBA}{Q3QzlaG3QGazOwtUl7N75Q}{10.9.126.87}{10.9.126.87:9300} committed version [4] source [zen-disco-node-join[{es-data-0}{SAOhUiLiRkazskZ_TC6EBQ}{qirmfVJBTjSBQtHZnz-QZw}{10.9.126.88}{10.9.126.88:9300}]]])
[2018-10-21T07:48:21,984][INFO ][o.e.c.s.ClusterApplierService] [es-master-594b58b86c-bj7g7] added {{es-data-1}{fiv5Wh29TRWGPumm5ypJfA}{EXqKGSzIQquRyWRzxIOWhQ}{10.9.125.82}{10.9.125.82:9300},}, reason: apply cluster state (from master [master {es-master-594b58b86c-bj7g7}{1aFT97hQQ7yiaBc2CYShBA}{Q3QzlaG3QGazOwtUl7N75Q}{10.9.126.87}{10.9.126.87:9300} committed version [5] source [zen-disco-node-join[{es-data-1}{fiv5Wh29TRWGPumm5ypJfA}{EXqKGSzIQquRyWRzxIOWhQ}{10.9.125.82}{10.9.125.82:9300}]]])
[2018-10-21T07:50:51,245][INFO ][o.e.c.s.ClusterApplierService] [es-master-594b58b86c-bj7g7] added {{es-client-69b84b46d8-v5pj2}{MMjA_tlTS7ux-UW44i0osg}{rOE4nB_jSmaIQVDZCjP8Rg}{10.9.125.83}{10.9.125.83:9300},}, reason: apply cluster state (from master [master {es-master-594b58b86c-bj7g7}{1aFT97hQQ7yiaBc2CYShBA}{Q3QzlaG3QGazOwtUl7N75Q}{10.9.126.87}{10.9.126.87:9300} committed version [6] source [zen-disco-node-join[{es-client-69b84b46d8-v5pj2}{MMjA_tlTS7ux-UW44i0osg}{rOE4nB_jSmaIQVDZCjP8Rg}{10.9.125.83}{10.9.125.83:9300}]]])
[2018-10-21T07:50:58,964][INFO ][o.e.c.s.ClusterApplierService] [es-master-594b58b86c-bj7g7] added {{es-client-69b84b46d8-kr7j4}{gGC7F4diRWy2oM1TLTvNsg}{IgI6g3iZT5Sa0HsFVMpvvw}{10.9.124.82}{10.9.124.82:9300},}, reason: apply cluster state (from master [master {es-master-594b58b86c-bj7g7}{1aFT97hQQ7yiaBc2CYShBA}{Q3QzlaG3QGazOwtUl7N75Q}{10.9.126.87}{10.9.126.87:9300} committed version [7] source [zen-disco-node-join[{es-client-69b84b46d8-kr7j4}{gGC7F4diRWy2oM1TLTvNsg}{IgI6g3iZT5Sa0HsFVMpvvw}{10.9.124.82}{10.9.124.82:9300}]]])
```

leading master pod的日志清楚地描述了每个节点何时添加到集群。 这在调试问题时非常有用。

部署完所有组件后，我们应验证以下内容：

1. 在kubernetes集群内部使用ubuntu容器进行Elasticsearch部署的验证。

```shell
root$ kubectl run my-shell --rm -i --tty --image ubuntu -- bash
root@my-shell-68974bb7f7-pj9x6:/# curl http://elasticsearch.elasticsearch:9200/_cluster/health?pretty
{
"cluster_name" : "my-es",
"status" : "green",
"timed_out" : false,
"number_of_nodes" : 7,
"number_of_data_nodes" : 2,
"active_primary_shards" : 0,
"active_shards" : 0,
"relocating_shards" : 0,
"initializing_shards" : 0,
"unassigned_shards" : 0,
"delayed_unassigned_shards" : 0,
"number_of_pending_tasks" : 0,
"number_of_in_flight_fetch" : 0,
"task_max_waiting_in_queue_millis" : 0,
"active_shards_percent_as_number" : 100.0
}
```

2. 在kubernetes集群外部使用GCP内部LoadBalancer IP(这里是10.9.120.8)进行Elasticsearch部署的验证。

```shell
root$ curl http://10.9.120.8:9200/_cluster/health?pretty
{
"cluster_name" : "my-es",
"status" : "green",
"timed_out" : false,
"number_of_nodes" : 7,
"number_of_data_nodes" : 2,
"active_primary_shards" : 0,
"active_shards" : 0,
"relocating_shards" : 0,
"initializing_shards" : 0,
"unassigned_shards" : 0,
"delayed_unassigned_shards" : 0,
"number_of_pending_tasks" : 0,
"number_of_in_flight_fetch" : 0,
"task_max_waiting_in_queue_millis" : 0,
"active_shards_percent_as_number" : 100.0
}
```

3. ES-Pods的Anti-Affinity规则验证。

```shell
root$ kubectl -n elasticsearch get pods -o wide 
NAME                         READY     STATUS    RESTARTS   AGE       IP            NODE
es-client-69b84b46d8-kr7j4   1/1       Running   0          10m       10.8.14.52   gke-cluster1-pool1-d2ef2b34-t6h9
es-client-69b84b46d8-v5pj2   1/1       Running   0          10m       10.8.15.53   gke-cluster1-pool1-42b4fbc4-cncn
es-data-0                    1/1       Running   0          12m       10.8.16.58   gke-cluster1-pool1-4cfd808c-kpx1
es-data-1                    1/1       Running   0          12m       10.8.15.52   gke-cluster1-pool1-42b4fbc4-cncn
es-master-594b58b86c-9jkj2   1/1       Running   0          18m       10.8.15.51   gke-cluster1-pool1-42b4fbc4-cncn
es-master-594b58b86c-bj7g7   1/1       Running   0          18m       10.8.16.57   gke-cluster1-pool1-4cfd808c-kpx1
es-master-594b58b86c-lfpps   1/1       Running   0          18m       10.8.14.51   gke-cluster1-pool1-d2ef2b34-t6h9
```

请注意，同一节点上没有2个类似的pod。 这可以在节点发生故障时确保HA。

### Scaling相关注意事项

我们可以根据CPU阈值为client节点部署autoscalers。 Client节点的HPA示例可能如下所示：

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: es-client
  namespace: elasticsearch
spec:
  maxReplicas: 5
  minReplicas: 2
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: es-client
targetCPUUtilizationPercentage: 80
```

每当autoscaler启动时，我们都可以通过观察任何master pod的日志来观察添加到集群中的新client节点pod。

对于Data Node Pod，我们必须使用K8 Dashboard或GKE控制台增加副本数量。 新创建的data节点将自动添加到集群中，并开始从其他节点复制数据。

Master Node Pod不需要自动扩展，因为它们只存储集群状态信息，但是如果要添加更多data节点，请确保集群中没有偶数个master节点，同时环境变量NUMBER_OF_MASTERS也需要相应调整。

### 部署Kibana和ES-HQ

Kibana是一个可视化ES数据的简单工具，ES-HQ有助于管理和监控Elasticsearch集群。 对于我们的Kibana和ES-HQ部署，我们记住以下事项：

- 我们提供ES-Cluster的名称作为docker镜像的环境变量
- 访问Kibana/ES-HQ部署的服务仅在我们组织内部，即不创建公共IP。 我们使用GCP内部负载均衡。

#### Kibana部署

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: elasticsearch
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: es-kibana
  namespace: elasticsearch
  labels:
    component: elasticsearch
    role: kibana
spec:
  replicas: 1
  template:
    metadata:
      labels:
        component: elasticsearch
        role: kibana
    spec:
      containers:
      - name: es-kibana
        image: docker.elastic.co/kibana/kibana-oss:6.2.2
        env:
        - name: CLUSTER_NAME
          value: my-es
        - name: ELASTICSEARCH_URL
          value: http://elasticsearch:9200
        resources:
          limits:
            cpu: 0.5
        ports:
        - containerPort: 5601
          name: http
---
apiVersion: v1
kind: Service
metadata:
  name: kibana
  annotations:
    cloud.google.com/load-balancer-type: "Internal"
  namespace: elasticsearch
  labels:
    component: elasticsearch
    role: kibana
spec:
  selector:
    component: elasticsearch
    role: kibana
  ports:
  - name: http
    port: 80
    targetPort: 5601
    protocol: TCP
  type: LoadBalancer
```

#### ES-HQ部署

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: elasticsearch
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: es-hq
  namespace: elasticsearch
  labels:
    component: elasticsearch
    role: hq
spec:
  replicas: 1
  template:
    metadata:
      labels:
        component: elasticsearch
        role: hq
    spec:
      containers:
      - name: es-hq
        image: elastichq/elasticsearch-hq:release-v3.4.0
        env:
        - name: HQ_DEFAULT_URL
          value: http://elasticsearch:9200
        resources:
          limits:
            cpu: 0.5
        ports:
        - containerPort: 5000
          name: http
---
apiVersion: v1
kind: Service
metadata:
  name: hq
  annotations:
    cloud.google.com/load-balancer-type: "Internal"
  namespace: elasticsearch
  labels:
    component: elasticsearch
    role: hq
spec:
  selector:
    component: elasticsearch
    role: hq
  ports:
  - name: http
    port: 80
    targetPort: 5000
    protocol: TCP
  type: LoadBalancer
```

我们可以使用新创建的Internal LoadBalancers访问这两个服务。

```shell
root$ kubectl -n elasticsearch get svc -l role=kibana
NAME      TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
kibana    LoadBalancer   10.9.121.246   10.9.120.10   80:31400/TCP   1m
root$ kubectl -n elasticsearch get svc -l role=hq
NAME      TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
hq        LoadBalancer   10.9.121.150   10.9.120.9    80:31499/TCP   1m
```

##### Kibana Dashboard `http://<External-Ip-Kibana-Service>/app/kibana#/home?_g=()`

![1*x_ohXVXlQDEAb0DTuOf3Og.png](https://cdn-images-1.medium.com/max/1600/1*x_ohXVXlQDEAb0DTuOf3Og.png)

##### ElasticHQ Dasboard `http://<External-Ip-ES-Hq-Service>/#!/clusters/my-es`

![1*cP7_RzAAI7cEeRJbKFQzDg.png](https://cdn-images-1.medium.com/max/1600/1*cP7_RzAAI7cEeRJbKFQzDg.png)

ES是最广泛使用的分布式搜索和分析系统之一，当与Kubernetes结合使用时，将消除有关扩展和HA的关键问题。 此外，使用Kubernetes部署新的ES群集需要时间。 我希望这个博客对你有用，我真的很期待改进的建议。 随意评论或联系[LinkedIn](https://www.linkedin.com/in/vaibhavthakur/)。





