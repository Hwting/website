---
title: "Scaling Mongodb on Kubernetes"
date: 2018-10-31T11:54:52+08:00
draft: true
banner: "https://cdn-images-1.medium.com/max/1600/1*JR_8ybLEuRw6ZnwKtcCMPA.png"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "文章摘要"
tags: ["Kubernetes", "MongoDB", "StatefulSets"]
categories: ["Kubernetes"]
---

# 如何在 Kubernetes 上扩展 MongoDB

[原文](https://medium.com/@thakur.vaibhav23/scaling-mongodb-on-kubernetes-32e446c16b82)

Kubernetes 主要用于无状态应用程序。 但是，在 1.3 版本中引入了 PetSets，之后它们演变为 StatefulSets。 官方文档将 StatefulSets 描述为“StatefulSets 旨在与有状态应用程序和分布式系统一起使用”。

对此最好的用例之一是对数据存储服务进行编排，例如 MongoDB，ElasticSearch，Redis，Zookeeper 等。

我们可以把 StatefulSets 的特性归纳如下：

1. 有序索引 Pods
2. 稳定的网络 ID
3. 有序并行的 Pods 管理
4. 滚动更新

这些细节可以在[这里](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/)找到。

StatefulSets 的一个非常明显的特征是提供稳定网络 ID，与[headless-services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)一起使用时，功能可以更加强大。

我们不花费很多时间在 Kubernetes 文档中随时可以查看的信息，让我们专注于运行和扩展 MongoDB 集群。

您需要一个可以运行的 Kubernetes 群集并启用 RBAC（推荐）。 在本教程中，我将使用 GKE 集群，但是，AWS EKS 或 Microsoft 的 AKS 或 Kops 管理的 K8S 也是可行的替代方案。

我们将为 MongoDB 集群部署以下组件:

1. 配置 HostVM 的 Daemon Set
2. Mongo Pods 的 Service Account 和 ClusterRole Binding
3. 为 Pods 提供永久性存储 SSDs 的 Storage Class
4. 访问 Mongo 容器的 Headless Service
5. Mongo Pods Stateful Set
6. GCP Internal LB: 从 kubernetes 集群外部访问 MongoDB(可选)
7. 使用 Ingress 访问 Pod(可选)

值得注意的是，每个 MongoDB Pod 都会运行一个 sidecar，以便动态配置副本集。Sidecar 每 5 秒检查一次新成员。

### Daemon Set for HostVM Configuration:

```yaml
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: hostvm-configurer
  labels:
    app: startup-script
spec:
  template:
    metadata:
      labels:
        app: startup-script
    spec:
      hostPID: true
      containers:
        - name: hostvm-configurer-container
          image: gcr.io/google-containers/startup-script:v1
          securityContext:
            privileged: true
          env:
            - name: STARTUP_SCRIPT
              value: |
                #! /bin/bash
                set -o errexit
                set -o pipefail
                set -o nounset

                # Disable hugepages
                echo 'never' > /sys/kernel/mm/transparent_hugepage/enabled
                echo 'never' > /sys/kernel/mm/transparent_hugepage/defrag
```

### Configuration for ServiceAccount, Storage Class, Headless SVC and StatefulSet:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mongo
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mongo
  namespace: mongo
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: mongo
subjects:
  - kind: ServiceAccount
    name: mongo
    namespace: mongo
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
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
apiVersion: v1
kind: Service
metadata:
  name: mongo
  namespace: mongo
  labels:
    name: mongo
spec:
  ports:
    - port: 27017
      targetPort: 27017
  clusterIP: None
  selector:
    role: mongo
---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: mongo
  namespace: mongo
spec:
  serviceName: mongo
  replicas: 3
  template:
    metadata:
      labels:
        role: mongo
        environment: staging
        replicaset: MainRepSet
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: replicaset
                      operator: In
                      values:
                        - MainRepSet
                topologyKey: kubernetes.io/hostname
      terminationGracePeriodSeconds: 10
      serviceAccountName: mongo
      containers:
        - name: mongo
          image: mongo
          command:
            - mongod
            - "--wiredTigerCacheSizeGB"
            - "0.25"
            - "--bind_ip"
            - "0.0.0.0"
            - "--replSet"
            - MainRepSet
            - "--smallfiles"
            - "--noprealloc"
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongo-persistent-storage
              mountPath: /data/db
          resources:
            requests:
              cpu: 1
              memory: 2Gi
        - name: mongo-sidecar
          image: cvallance/mongo-k8s-sidecar
          env:
            - name: MONGO_SIDECAR_POD_LABELS
              value: "role=mongo,environment=staging"
            - name: KUBE_NAMESPACE
              value: "mongo"
            - name: KUBERNETES_MONGO_SERVICE_NAME
              value: "mongo"
  volumeClaimTemplates:
    - metadata:
        name: mongo-persistent-storage
        annotations:
          volume.beta.kubernetes.io/storage-class: "fast"
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast
        resources:
          requests:
            storage: 10Gi
```

关键点：

1. 应该使用适当的环境变量仔细配置 Mongo 的 Sidecar，以及为 pod 提供的标签，和为 deployment 和 service 的命名空间。 有关 sidecar 容器的详细信息，请点击[此处](https://github.com/cvallance/mongo-k8s-sidecar)。
2. 默认缓存大小的指导值是：“50％的 RAM 减去 1GB，或 256MB”。 鉴于所请求的内存量为 2GB，此处的 WiredTiger 缓存大小已设置为 256MB。
3. Inter-Pod Anti-Affinity 确保在同一个工作节点上不会安排 2 个 Mongo Pod，从而使其能够适应节点故障。 此外，建议将节点保留在不同的可用区中，以便集群能够抵御区域故障。
4. 当前部署的 Service Account 具有管理员权限。 但是，它应该仅限于 DB 的命名空间。

上面提到的两个配置文件也可以在[这里](https://github.com/Thakurvaibhav/k8s/tree/master/databases/mongodb)找到。

### 部署 MongoDB 集群

```shell
kubectl apply -f configure-node.yml
kubectl apply -f mongo.yml
```

你可以通过以下命令查看所有组件状况：

```shell
kubectl -n mongo get all
```

```shell
root$ kubectl -n mongo get all
NAME                 DESIRED   CURRENT   AGE
statefulsets/mongo   3         3         3m
NAME         READY     STATUS    RESTARTS   AGE
po/mongo-0   2/2       Running   0          3m
po/mongo-1   2/2       Running   0          2m
po/mongo-2   2/2       Running   0          1m
NAME        TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
svc/mongo   ClusterIP   None         <none>        27017/TCP   3m
```

如您所见，该服务没有 Cluster-IP，也没有 External-IP，它是 Headless 服务。 此服务将直接解析为 StatefulSets 的 Pod-IP。

让我们来验证一下 DNS 解析。 我们在集群中启动了一个交互式 shell：

```shell
kubectl run my-shell --rm -i --tty --image ubuntu -- bash
root@my-shell-68974bb7f7-cs4l9:/# dig mongo.mongo +search +noall +answer
; <<>> DiG 9.11.3-1ubuntu1.1-Ubuntu <<>> mongo.mongo +search +noall +answer
;; global options: +cmd
mongo.mongo.svc.cluster.local. 30 IN A 10.56.7.10
mongo.mongo.svc.cluster.local. 30 IN A 10.56.8.11
mongo.mongo.svc.cluster.local. 30 IN A 10.56.1.4
```

服务的 DNS 规则是<服务名称>.<服务的命名空间>，因此，在我们的例子中看到的是 mongo.mongo。

IPs（10.56.6.17,10.56.7.10,10.56.8.11）是我们的 Mongo StatefulSets 的 Pod IPs。 这可以通过在集群内部运行 nslookup 来测试。

```shell
root@my-shell-68974bb7f7-cs4l9:/# nslookup 10.56.6.17
17.6.56.10.in-addr.arpa name = mongo-0.mongo.mongo.svc.cluster.local.
root@my-shell-68974bb7f7-cs4l9:/# nslookup 10.56.7.10
10.7.56.10.in-addr.arpa name = mongo-1.mongo.mongo.svc.cluster.local.
root@my-shell-68974bb7f7-cs4l9:/# nslookup 10.56.8.11
11.8.56.10.in-addr.arpa name = mongo-2.mongo.mongo.svc.cluster.local.
```

如果您的应用程序部署在 K8 的群集中，那么它可以通过以下方式访问节点:

```
Node-0: mongo-0.mongo.mongo.svc.cluster.local:27017
Node-1: mongo-1.mongo.mongo.svc.cluster.local:27017
Node-2: mongo-2.mongo.mongo.svc.cluster.local:27017
```

如果要从集群外部访问 mongo 节点，你可以为每个 pod 部署内部负载平衡或使用 Ingress Controller（如 NGINX 或 Traefik）创建一个内部 Ingress。

### GCP Internal LB SVC Configuration (可选)

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    cloud.google.com/load-balancer-type: Internal
  name: mongo-0
  namespace: mongo
spec:
  ports:
    - port: 27017
      targetPort: 27017
  selector:
    statefulset.kubernetes.io/pod-name: mongo-0
  type: LoadBalancer
```

为 mongo-1 和 mongo-2 也部署 2 个此类服务。

您可以将内部负载均衡的 IP 提供给 MongoClient URI。

```shell
root$ kubectl -n mongo get svc
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
mongo     ClusterIP      None            <none>        27017/TCP         15m
mongo-0   LoadBalancer   10.59.252.157   10.20.20.2    27017:30184/TCP   9m
mongo-1   LoadBalancer   10.59.252.235   10.20.20.3    27017:30343/TCP   9m
mongo-2   LoadBalancer   10.59.254.199   10.20.20.4    27017:31298/TCP   9m
```

mongo-0/1/2 的外部 IP 是新创建的 TCP 负载均衡器的 IP。 这些是您的子网或对等网络，如果有的话。

### 通过 Ingress 访问 Pods(可选)

也可以使用诸如 Nginx 之类的 Ingress Controller 来定向到 Mongo StatefulSets 的流量。 确保 ingress 服务是内部服务，而不是通过 PublicIP 公开。 Ingress 对象的配置看起来像这样：

```yaml

---
spec:
  rules:
    - host: mongo.example.com
      http:
        paths:
          - path: "/mongo-0"
            backend:
              hostNames:
                - mongo-0
              serviceName: mongo # There is no extra service. This is the headless service.
              servicePort: "27017"
```

请务必注意，您的应用程序至少应该知道一个当前处于启动状态的 mongo 节点，这样可以发现所有其他节点。

我在本地 mac 上使用 Robo 3T 作为 mongo 客户端。 连接到其中一个节点后并运行 rs.status()，您可以查看副本集的详细信息，并检查是否已配置其他 2 个 Pods 并自动连接到副本集。

rs.status()查看副本集名称和成员个数：

![1*RCYiGWoLvewtHXBTRpcZ3Q.jpeg](https://cdn-images-1.medium.com/max/1600/1*RCYiGWoLvewtHXBTRpcZ3Q.jpeg)

每个成员都可以看到 FQDN 和状态。 此 FQDN 只能从群集内部访问。

![1*xvYbCWgJmcjd6ksS0dhifw.jpeg](https://cdn-images-1.medium.com/max/1600/1*xvYbCWgJmcjd6ksS0dhifw.jpeg)

每个 secondary 成员正在同步到 mongo-0，mongo-0 是当前的 primary。

![1*rrTIXV7zH58YawD0uW5eGw.jpeg](https://cdn-images-1.medium.com/max/1600/1*rrTIXV7zH58YawD0uW5eGw.jpeg)

现在我们扩展 mongo Pods 的 Stateful Set 以检查新的 mongo 容器是否被添加到 ReplicaSet。

```shell
root$ kubectl -n mongo scale statefulsets mongo --replicas=4
statefulset "mongo" scaled
root$ kubectl -n mongo get pods -o wide
NAME      READY     STATUS    RESTARTS   AGE       IP           NODE
mongo-0   2/2       Running   0          25m       10.56.6.17   gke-k8-demo-demo-k8-pool-1-45712bb7-vfqs
mongo-1   2/2       Running   0          24m       10.56.7.10   gke-k8-demo-demo-k8-pool-1-c6901f2e-trv5
mongo-2   2/2       Running   0          23m       10.56.8.11   gke-k8-demo-demo-k8-pool-1-c7622fba-qayt
mongo-3   2/2       Running   0          3m        10.56.1.4    gke-k8-demo-demo-k8-pool-1-85308bb7-89a4
```

可以看出，所有四个 pod 都部署到不同的 GKE 节点，因此我们的 Pod-Anti Affinity 策略工作正常。

扩展操作还将自动提供持久卷，该卷将充当新 pod 的数据目录。

```shell
root$ kubectl -n mongo get pvc
NAME                               STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongo-persistent-storage-mongo-0   Bound     pvc-337fb7d6-9f8f-11e8-bcd6-42010a940024   11G        RWO            fast           49m
mongo-persistent-storage-mongo-1   Bound     pvc-53375e31-9f8f-11e8-bcd6-42010a940024   11G        RWO            fast           49m
mongo-persistent-storage-mongo-2   Bound     pvc-6cee0f97-9f8f-11e8-bcd6-42010a940024   11G        RWO            fast           48m
mongo-persistent-storage-mongo-3   Bound     pvc-3e89573f-9f92-11e8-bcd6-42010a940024   11G        RWO            fast           28m
```

要检查名为 mongo-3 的 pod 是否已添加到副本集，我们将在同一节点上再次运行 rs.status()并观察其差异。

对于同一个的 Replicaset，成员数现在为 4。

![1*NeZZY-RBs0EzUuVqam_zaQ.jpeg](https://cdn-images-1.medium.com/max/1600/1*NeZZY-RBs0EzUuVqam_zaQ.jpeg)

新添加的成员遵循与先前成员相同的 FQDN 方案，并且还与同一主节点同步：

![1*HW0lDpjmf9wT6bLaA0xImg.jpeg](https://cdn-images-1.medium.com/max/1600/1*HW0lDpjmf9wT6bLaA0xImg.jpeg)

### 进一步的考虑

1. 给 Mongo Pod 的 Node Pool 打上合适的 label 并确保在 StatefulSets 和 HostVM 配置的 DaemonSets 的 Spec 中指定适当的 Node Affinity 会很有帮助。 这是因为 DaemonSet 将调整主机操作系统的一些参数，并且这些设置应仅限于 MongoDB Pod。 没有这些设置，对其他应用程序可能会更好。
2. 在 GKE 中给 Node Pool 打 Label 非常容易，可以直接从 GCP 控制台进行。
3. 虽然我们在 Pod 的 Spec 中指定了 CPU 和内存限制，但我们也可以考虑部署 VPA（Vertical Pod Autoscaler）。
4. 可以通过实施网络策略或服务网格（如 Istio）来控制从集群内部到我们的数据库的流量。

如果你已经看到这里，我相信你已经浏览了整个博文。 我试图整理很多分散的信息并将其作为一个整体呈现。 我的目标是为您提供足够的信息，以便开始使用 Kubernetes 上的 Stateful Sets，并希望你们中的许多人觉得它很有用。 我们非常欢迎您提出反馈、意见或建议。:)
