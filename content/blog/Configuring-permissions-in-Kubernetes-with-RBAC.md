---
title: "Configuring Permissions in Kubernetes With RBAC"
date: 2018-11-04T16:14:23+08:00
draft: false
banner: "https://cdn-images-1.medium.com/max/1600/1*nywxB5PdmQq8zmB_TnMTbQ.jpeg"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "文章摘要"
tags: ["Kubernetes", "RBAC"]
categories: ["Kubernetes"]
---

# Configuring permissions in Kubernetes with RBAC

[原文](https://medium.com/containerum/configuring-permissions-in-kubernetes-with-rbac-a456a9717d5d)

确保控制谁有权访问您的信息系统以及用户可以访问的内容是身份和访问管理系统的目标。 它是安全管理的基本流程之一，应该彻底解决。

在 Kubernetes 中，身份和用户管理未集成在平台中，应由外部 IAM 平台（如 Keycloak，Active Directory，Google 的 IAM 等）进行管理。但是，身份验证和授权由 Kubernetes 处理。

在本文中，我们将重点关注 Kubernetes 中 IAM 的授权方面，更具体地说，关于如何使用基于角色的访问控制模型确保用户对相应的资源具有正确的权限。

## 先决条件

RBAC 是 Kubernetes 1.8 的稳定功能。 在本文中，我们假设您使用的是 Kubernetes 1.9+。 我们还假设您的集群中已通过 Kubernetes API 服务器中的`--authorization-mode = RBAC`选项启用了 RBAC。 您可以通过执行命令`kubectl api-versions`来检查这一点; 如果启用了 RBAC，您应该看到 API 版本`.rbac.authorization.k8s.io/v1`。

## RBAC 核心内容概览

Kubernetes 中的 RBAC 模型基于三个要素：

- Roles：定义每个 Kubernetes 资源类型的权限
- Subjects：用户（人或机器用户）或用户组
- RoleBindings：定义哪些 subjects 具有哪些 roles

让我们来探索这些元素是如何工作的。

在下面的示例中，您可以看到一个允许在命名空间“mynamespace”中获取，列出和监视 pod 的角色：

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: mynamespace
  name: example-role
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
```

要为用户提供上面角色中描述的权限，必须创建一个 RoleBinding。 在下面的示例中，RoleBinding“example-rolebinding”将角色“example-role”绑定到用户“example-user”：

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
  - kind: User
    name: example-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```

应该注意的是，Roles 和 RoleBindings 是命名空间的，这意味着只能为与 Role 和 RoleBinding 位于同一命名空间的 Kubernetes 资源提供权限。 此外，毫无疑问，RoleBinding 只能引用其命名空间中存在的角色。

## Roles, ClusterRoles, RoleBindings 和 ClusterRoleBindings

在前面的示例中，我们使用了 Roles 和 RoleBindings。 但是，也有可能使用 ClusterRoles 和 ClusterRoleBindings，它们在以下情况下很有用：

- 为非命名空间资源（如节点）授予权限
- 为群集的所有命名空间中的资源授予权限
- 为`/healthz`等非资源端点授予权限

Roles 和 RoleBindings 的集群版本的定义与非集群版本非常相似。 如果我们采用前面的示例并对其进行调整，我们将具有以下定义。

ClusterRole：

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrole
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
```

ClusterRoleBinding:

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrolebinding
subjects:
  - kind: User
    name: example-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: example-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

## 如何在 Roles 和 ClusterRoles 中启用权限

在第一个示例中，我们向用户授予了获取，观察和列出 pod 的基本权限。 让我们探索我们对不同资源和权限的其他可能性。

在下面的示例中，Role 允许使用 Deployment 资源执行任何操作：

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: mynamespace
  name: example-role-2
rules:
  - apiGroups: ["extensions", "apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

请注意，在这种情况下，apiGroups 字段已填入 Deployment 的 API 组。 根据 Kubernetes 版本，部署资源可在 API `apps/v1`或`extensions/v1beta2`中找到; API 组是斜杠之前的部分。

我们可以在角色中定义多个规则，如下例所示：

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["batch", "extensions"]
    resources: ["jobs"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

在这种情况下，我们根据目标资源是 Pod 还是 Job 授予不同的权限。

我们还可以通过名称来定位资源，如下例所示：

```yaml
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["my-config"]
    verbs: ["get"]
```

## 如何将 Subject 绑定到 Role 或 ClusterRole

在第一个示例中，我们已经了解了如何将人类用户绑定到角色。 但是，也可以绑定服务帐户 service account（非人类用户）或一组人类用户和/或服务帐户 service account。

在下面的示例中，RoleBinding `example-rolebinding`绑定 ServiceAccount `ex-sto`到 Role `example-role`：

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
  - kind: ServiceAccount
    name: example-sa
    namespace: mynamespace
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```

您可以使用以下命令创建 ServiceAccount：

```shell
kubectl create serviceaccount example-sa --namespace mynamespace
```

在之前的 RoleBinding 定义中，我们还可以替换 subjects 以绑定 group。 在下面的示例中，我们绑定 frontend-adminsgroup：

```yaml
subjects:
  - kind: Group
    name: "frontend-admins"
    apiGroup: rbac.authorization.k8s.io
```

另一种可能性是绑定服务帐户组(service account group)。 在这里，我们绑定 mynamespace 命名空间中的所有服务帐户(service account)：

```yaml
subjects:
  - kind: Group
    name: system:serviceaccounts:mynamespace
    apiGroup: rbac.authorization.k8s.io
```

或者是集群中所有的 service account：

```yaml
subjects:
  - kind: Group
    name: system:serviceaccounts
    apiGroup: rbac.authorization.k8s.io
```

## 关于 Kubernetes 的 RBAC 的最后一件事

我们已经了解了如何使用基于角色的访问控制模型向用户或服务账户(service account)授予权限。 这是在 Kubernetes 中实现授权的有效方式，它可能是最受欢迎的方法，但它不是唯一可用的模型：您还可以使用其他模型，如 ABAC（基于属性的访问控制），节点授权模型和 Webhook 模式。 我们将在后续文章中描述这些内容，以及 Kubernetes 中的另一个 IAM 功能：身份验证。
