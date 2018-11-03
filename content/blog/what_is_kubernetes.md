---
title: "Kubernetes部署实操教程"
date: 2018-11-04T01:02:54+08:00
draft: false
banner: "https://uploads.toptal.io/blog/image/127205/toptal-blog-image-1537248175479-d7f23b2548ab0238fde7a730937e7be4.png"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "在本文中，我将向您介绍容器，解释 Kubernetes，并教您如何使用 CircleCI 将应用程序容器化和部署到 Kubernetes 集群。"
tags: ["Kubernetes"]
categories: ["Kubernetes"]
---

# Kubernetes 部署实操教程

[原文](https://www.toptal.com/kubernetes/what-is-kubernetes)

之前，我们基本都是单体 web 应用程序：大型的代码库，随着新的功能和特性不断发展，最后它们都会变成巨大的，缓慢移动的，难以管理的巨人。 现在，越来越多的开发人员，架构师和 DevOps 专家认为，使用微服务比使用大型单体应用更好。 通常，使用基于微服务的体系结构意味着将您的单体应用分成至少两个应用程序：前端应用程序和后端应用程序（API）。 在决定使用微服务之后，出现了一个问题：在什么环境下运行微服务更好？ 我应该选择什么来使我的服务稳定，易于管理和部署？ 简短的回答是：使用 Docker！

在本文中，我将向您介绍容器，解释 Kubernetes，并教您如何使用 CircleCI 将应用程序容器化和部署到 Kubernetes 集群。

Docker?什么是 Docker?

Docker 是一款旨在让 DevOps（和您的生活）更轻松的工具。 使用 Docker，开发人员可以在容器中创建，部署和运行应用程序。 容器允许开发人员使用所需的所有部件（例如库和其他依赖项）打包应用程序，并将其作为一个包发布出去。

![toptal-blog-image-1537248149370-5173639a83fafe346834d5d29c851be0.png](https://uploads.toptal.io/blog/image/127203/toptal-blog-image-1537248149370-5173639a83fafe346834d5d29c851be0.png)

使用容器，开发人员可以轻松将镜像（重新）部署到任何操作系统。 只需安装 Docker，执行命令，您的应用程序即可启动并运行。 哦，不要担心主机操作系统中新版本库的任何不一致。 此外，您可以在同一主机上启动很多容器 - 不管是相同的应用程序还是其他应用程序，都没关系。

看起来 Docker 是一个很棒的工具。 但是我应该如何以及在何处启动容器？

运行容器的方式有很多选择：

- AWS Elastic Container Service（AWS Fargate 或具有水平和垂直自动伸缩的预留实例）;
- Azure 或 Google Cloud 中具有预定义 Docker 镜像的云实例（包含模板，实例组和自动缩放）;
- 在您自己的服务器上;
- 当然还有 Kubernetes！ Kubernetes 是 2014 年由 Google 工程师专门为虚拟化和容器创建的。

Kubernetes？ 那是什么？

Kubernetes 是一个开源系统，允许您运行容器，管理容器，自动化部署，扩展部署，创建和配置 Ingress，部署无状态或有状态应用程序以及许多其他内容。 基本上，您可以启动一个或多个实例来安装 Kubernetes，将其作为 Kubernetes 集群进行操作。 然后获取 Kubernetes 集群的 API 端点，配置 kubectl（管理 Kubernetes 集群的工具），Kubernetes 即可投入使用。

那我为什么要用 Kubernetes 呢？

使用 Kubernetes，您可以最大限度地利用计算资源。 通过 Kubernetes，您将成为您的船（基础设施）的船长，Kubernetes 将为您扬帆。 使用 Kubernetes，您的服务将是高可用的。 最重要的是，通过 Kubernetes，您将节省大量资金。

看起来很有前途，特别是如果它会省钱！ 让我们来谈谈它！

Kubernetes 日复一日地更加受欢迎。 让我们更深入地研究一下这幕后的内容。

![toptal-blog-image-1537248162316-c761b80b0f6eb33884e18bcde9c42a98.png](https://uploads.toptal.io/blog/image/127204/toptal-blog-image-1537248162316-c761b80b0f6eb33884e18bcde9c42a98.png)

（译者注：上图有个小错误，kubectl 写成了 kubecti。）

Kubernetes 是整个系统的名称，但与您的汽车一样，有许多小部件可以完美地彼此协同工作以使 Kubernetes 发挥其各种作用。 让我们来了解它们分别是什么。

主节点（Master Node） - 整个 Kubernetes 集群的控制面板。 主节点的组件可以在群集中的任何节点上运行。 关键组成部分是：

- API Server：所有 REST 命令的入口点，是可以让用户可以访问的主节点的唯一组件。
- Datastore: Kubernetes 群集使用的强大，一致且高可用的键值存储。
- Scheduler：监视新创建的 pod 并将它们分配给节点。 pod 和 services 部署到节点上主要由 scheduler 来控制。
- Controller manager: 管理着处理集群中日常任务的所有控制器。
- Worker Node：主要的节点代理，也称为 minion(旧版本 kubernetes 的叫法)。 Pod 在这里运行。 工作节点包含所有必要的服务，这些服务包括管理容器之间的网络，与主节点通信以及将资源分配给已调度容器等。
- Docker：运行在每个工作节点上，用来下载镜像和启动容器。
- Kubelet：监视 pod 的状态并确保容器已启动并运行。 它还与数据存储通信，获取有关服务的信息并记录新创建的服务的详细信息。
- Kube-proxy：单个工作节点上的服务的网络代理和负载均衡。 它负责流量路由。
- Kubectl：一个 CLI 工具，供用户与 Kubernetes API Server 进行通信。

那什么是 pods 和 services？

Pod 是 Kubernetes 集群中最小的单元，就像一座巨大建筑物墙上的一块砖。 pod 是一组需要一起运行并且可以共享资源的容器（Linux 名称空间，cgroups，IP 地址）。 Pod 的生命周期是非永久性的。

Service 是在多个 pod 之上的抽象，通常需要在它上面有层代理，以便其他服务通过虚拟 IP 地址与其通信。

### 一个简单部署例子

![toptal-blog-image-1537248175479-d7f23b2548ab0238fde7a730937e7be4.png](https://uploads.toptal.io/blog/image/127205/toptal-blog-image-1537248175479-d7f23b2548ab0238fde7a730937e7be4.png)

我将使用简单的 Ruby on Rails 应用程序和 GKE 作为运行 Kubernetes 的平台。 实际上，您可以在 AWS 或 Azure 中使用 Kubernetes，甚至可以在您自己的硬件中创建集群，或使用您 minikube 在本地运行 Kubernetes，所有的选项你都可以在这个页面[Setup - Kubernetes](https://kubernetes.io/docs/setup/)上找到。

这个 app 的源码你可以在[GitHub - d-kononov/simple-rails-app-in-k8s](https://github.com/d-kononov/simple-rails-app-in-k8s)里找到。

要创建新的 Rails 应用程序，请执行：

```shell
rails new blog
```

在`config/database.yml`文件中配置生产 MySQL 连接：

```yaml
production:
  adapter: mysql2
  encoding: utf8
  pool: 5
  port: 3306
  database: <%= ENV['DATABASE_NAME'] %>
  host: 127.0.0.1
  username: <%= ENV['DATABASE_USERNAME'] %>
  password: <%= ENV['DATABASE_PASSWORD'] %>
```

创建 Article model, controller, views, 和 migration, 请执行:

```shell
rails g scaffold Article title:string description:text
```

添加 Gems 到 Gemfile：

```
gem 'mysql2', '< 0.6.0', '>= 0.4.4'
gem 'health_check'
```

创建 Docker 镜像，请下载[Dockerfile](https://github.com/d-kononov/simple-rails-app-in-k8s/blob/master/Dockerfile)，并执行：

```shell
docker build -t REPO_NAME/IMAGE_NAME:TAG . && docker push REPO_NAME/IMAGE_NAME:TAG
```

是时候创建一个 Kubernetes 集群了。 打开 GKE 页面并创建 Kubernetes 集群。 创建集群后，单击“连接”按钮并复制命令 - 确保您已安装和配置了`gCloud CLI`工具（[如何](https://cloud.google.com/kubernetes-engine/docs/quickstart)）和 kubectl。 在 PC 上执行复制的命令并检查与 Kubernetes 集群的连接; 执行`kubectl cluster-info`。

该应用程序已准备好部署到 k8s 群集。 让我们创建一个 MySQL 数据库。 在 gCloud 控制台中打开 SQL 页面，为应用程序创建一个 MySQL 数据库实例。 实例准备就绪后，创建用户和数据库并复制实例连接名称。

此外，我们需要在`API和服务`页面中创建一个`service account`密钥，以便从 sidecar 容器访问 MySQL 数据库。 您可以在[此处](https://cloud.google.com/sql/docs/mysql/connect-kubernetes-engine#2_create_a_service_account)找到有关该流程的更多信息。 将下载的文件重命名为`service-account.json`。 我们稍后会用到这个文件。

我们准备将我们的应用程序部署到 Kubernetes，但首先，我们应该为我们的应用程序创建`secret` - 在 Kubernetes 中创建用于存储敏感数据的 secret 对象。 上传之前下载的`service-account.json`文件：

```shell
kubectl create secret generic mysql-instance-credentials \
                      --from-file=credentials.json=service-account.json
```

为应用程序创建 secrets：

```shell
kubectl create secret generic simple-app-secrets \
                      --from-literal=username=$MYSQL_PASSWORD \
                      --from-literal=password=$MYSQL_PASSWORD \
                      --from-literal=database-name=$MYSQL_DB_NAME \
                      --from-literal=secretkey=$SECRET_RAILS_KEY
```

请不要忘记替换必要的配置，或者记得正确地设置环境变量。

在创建 deployment 之前，让我们看一下[deployment.yaml](https://github.com/d-kononov/simple-rails-app-in-k8s/blob/master/-kubernetes/deployment.yaml)。 我把三个文件连成一个; 第一部分是一个 service，它将公开端口 80 并转发到端口 80 和到 3000 的所有连接。该 service 有一个 selector，service 通过它知道应该哪个 pod 应该转发连接。

该文件的下一部分是 deployment，它描述了将在 pod 中启动的部署策略容器，环境变量，资源，探针，每个容器的 mounts 以及其他信息。

最后一部分是 Horizontal Pod Autoscaler。 HPA 有一个非常简单的配置。 请记住，如果您未在部署部分中为容器设置资源，则 HPA 将无法运行。

您可以在 GKE 编辑页面中为 Kubernetes 集群配置 Vertical Autoscaler。 它也有一个非常简单的配置。

是时候将它发布到 GKE 集群了！ 首先，我们通过[rake-tasks-job.yaml](https://github.com/d-kononov/simple-rails-app-in-k8s/blob/master/-kubernetes/rake-tasks-job.yaml)进行部署。 执行：

```shell
kubectl apply -f rake-tasks-job.yaml
```

这个 job 对于 CI/CD 很有用。

```shell
kubectl apply -f deployment.yaml
```

上面这条命令用来创建 service, deployment 和 HPA。

然后通过`kubectl get pods -w`命令来检查你的 pods：

```
NAME                      READY     STATUS    RESTARTS   AGE
sample-799bf9fd9c-86cqf   2/2       Running   0          1m
sample-799bf9fd9c-887vv   2/2       Running   0          1m
sample-799bf9fd9c-pkscp   2/2       Running   0          1m
```

现在让我们为应用创建一个 Ingress：

1. 创建一个静态 IP：`gcloud compute addresses create sample-ip --global`
2. 创建 Ingress：`kubectl apply -f ingress.yaml`
3. 检查 Ingress 是否创建成功和查看它的 IP 地址：`kubectl get ingress -w`
4. 为你的应用创建域名或者子域名

### CI/CD

让我们使用 CircleCI 创建一个 CI/CD 管道。 实际上，使用 CircleCI 创建 CI/CD 管道很容易，但请记住，快而脏的未经过测试的全自动部署过程仅适用于小型项目，但请不要在严肃的项目上这样做。如果任何新代码在生产中出现问题，那么你将会损失巨大。 这就是为什么你应该考虑设计一个强大的部署过程，在完全推出之前启动 canary 任务，在 canary 启动后检查日志中的错误，等等。

目前，我们有一个简单的小项目，所以让我们创建一个完全自动化，无测试的 CI / CD 部署过程。 首先，您应该将 CircleCI 与您的代码仓库进行集成 - 您可以在此处[config.yml](https://github.com/d-kononov/simple-rails-app-in-k8s/blob/master/.circleci/config.yml)找到具体内容。 然后我们应该创建一个包含 CircleCI 系统指令的配置文件。 Config 看起来很简单。 要点是 GitHub 仓库中有两个分支：master 和 production。

1. 主分支用于开发，用于新代码。 当有人将新代码推送到主分支时，CircleCI 启动主分支构建和测试代码的工作流。
2. 生产分支用于将新版本部署到生产环境。 生产分支的工作流程如下：推送新代码（如果从主分支到生产开放 PR 则更好）以触发新的构建和部署过程; 在构建过程中，CircleCI 创建新的 Docker 镜像，将其推送到 GCR 并创建新的部署; 如果失败，CircleCI 将触发回滚过程。

在运行任何构建之前，您应该在 CircleCI 中配置项目。 在 API 和 GCloud 中的 service 页面中创建一个新的 service account，具有以下角色：完全访问 GCR 和 GKE，打开下载的 JSON 文件并复制内容，然后在 CircleCI 的项目设置中创建一个新的环境变量 GCLOUD_SERVICE_KEY 并将 service account 文件的内容粘贴为值。 此外，您需要创建下一个 env vars：GOOGLE_PROJECT_ID（您可以在 GCloud 控制台主页上找到它），GOOGLE_COMPUTE_ZONE（您的 GKE 集群的区域）和 GOOGLE_CLUSTER_NAME（GKE 集群名称）。

CircleCI 的最后一步（部署）如下所示：

```
kubectl patch deployment sample -p '{"spec":{"template":{"spec":{"containers":[{"name":"sample","image":"gcr.io/test-d6bf8/simple:'"$CIRCLE_SHA1"'"}]}}}}'
if ! kubectl rollout status deploy/sample; then
  echo "DEPLOY FAILED, ROLLING BACK TO PREVIOUS"
  kubectl rollout undo deploy/sample
  # Deploy failed -> notify slack
else
  echo "Deploy succeeded, current version: ${CIRCLE_SHA1}"
  # Deploy succeeded -> notify slack
fi
deployment.extensions/sample patched
Waiting for deployment "sample" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "sample" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "sample" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "sample" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "sample" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "sample" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "sample" rollout to finish: 2 of 3 updated replicas are available...
Waiting for deployment "sample" rollout to finish: 2 of 3 updated replicas are available...
deployment "sample" successfully rolled out
Deploy succeeded, current version: 512eabb11c463c5431a1af4ed0b9ebd23597edd9
```

### 总结

看起来创建新的 Kubernetes 集群的过程并不那么难！ CI/CD 自动部署过程也非常棒！

是! Kubernetes 太棒了！使用 Kubernetes，您的系统将更加稳定，易于管理，并使您成为系统的“船长”。更不用说，Kubernetes 对系统施加了一些魔法，并为您的营销提供了+100 分！

现在您已经掌握了基础知识，您可以进一步将其转换为更高级的配置。我计划在以后的文章中介绍更多内容，但与此同时，有一个挑战：使用位于集群内的有状态数据库（包括用于备份的 sidecar Pod）为您的应用程序创建一个强大的 Kubernetes 集群，在其中安装 Jenkins 用于 CI/CD 管道，让 Jenkins 使用 pod 作为构建的 slave。使用 certmanager 为您的 Ingress 添加/获取 SSL 证书。使用 Stackdriver 为您的应用程序创建监控和报警系统。

Kubernetes 非常棒，因为它很容易扩展，没有供应商锁定，并且，因为您为实例付费，所以您可以省钱。然而，不是每个人都是 Kubernetes 专家或者有时间建立一个新的集群。这里有一个可选方案：关于如何构建有效的初始部署管道并执行更少的操作任务，我的同事 Toptaler Amin Shah Gilani 使用 Heroku，GitLab CI 以及他能想到的其他自动化部署工具做了一套教程[CI/CD:如何部署有效的初始化部署管道](https://www.toptal.com/devops/effective-ci-cd-deployment-pipeline)。
