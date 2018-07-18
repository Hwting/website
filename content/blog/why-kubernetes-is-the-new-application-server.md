---
title: "为何说kubernetes是新一代的应用服务器(译)"
date: 2018-07-18T21:53:43+08:00
draft: false
banner: "https://developers.redhat.com/blog/wp-content/uploads/2018/05/Screenshot-2018-05-18-21.20.31.png"
author: "kelvinji2009"
authorlink: "https://kelvinji2009.github.io"
summary: "你有没有想过为什么你要使用容器部署你的多平台应用程序？这只是“跟随炒作”的问题吗？在本文中，我将要问一些挑衅性的问题，以说明为什么Kubernetes是新一代的应用服务器。"
tags: ["kubernetes","java","istio"]
categories: ["kubernetes"]
---

# 为何说kubernetes是新一代的应用服务器

[原文](https://developers.redhat.com/blog/2018/06/28/why-kubernetes-is-the-new-application-server/)

你有没有想过为什么你要使用容器部署你的多平台应用程序？这只是“跟随炒作”的问题吗？在本文中，我将要问一些挑衅性的问题，以说明为什么Kubernetes是新一代的应用服务器。

您可能已经注意到大多数语言都是被`interpreted`并使用`runtime`来执行您的源代码。 理论上，大多数Node.js，Python和Ruby代码可以轻松地从一个平台（Windows，Mac，Linux）迁移到另一个平台。 Java应用程序更进一步，通过将编译后的Java类转换为字节码，能够在具有JVM（Java虚拟机）的任何地方运行。

Java生态系统提供了一种标准格式，用于分发属于同一应用程序的所有Java类。 您可以将这些类打包为包含嵌入的前端，后端和库的JAR（Java归档），WAR（Web归档）和EAR（企业归档）。 那么我问你：为什么要使用容器来分发你的Java应用程序？ 难道它不应该在环境之间轻松移植吗？

从开发人员的角度回答这个问题并不总是显而易见的。 但请考虑一下您的开发环境以及由它与生产环境之间的差异引起的一些可能问题：

* 你使用的是Mac，Windows还是Linux？ 您是否遇到过与'\'vs'/'作为文件路径分隔符相关的问题？
* 你使用的是什么版本的JDK？ 你是否在开发中使用Java 10，但生产使用JRE 8？ 您是否遇到过JVM差异引入的错误？
* 你使用的是什么版本的应用服务器？ 生产环境是否使用相同的配置，安全补丁和库版本？
* 在生产部署期间，您是否遇到过由于驱动程序或数据库服务器的版本不同而导致的JDBC驱动程序问题，而在开发环境中却没有出现？
* 你有没有遇到当你要求应用服务器管理员创建一个数据源或一个JMS队列的时候，它却包含一个错误的字符？

上述所有问题都是由应用程序外部因素引起的，容器最重要的一点是您可以部署所有应用程序（例如，Linux发行版，JVM，应用程序服务器，库，配置以及您的最终应用程序）在预先构建好的容器中。 另外，执行一个内置了所有内容的容器比将代码移动到生产环境并尝试在不工作时解决差异要容易得多。 由于它很容易执行，因此将同一容器水平扩展多个副本也变得很容易。

## 强化你的应用

在容器变得非常流行之前，应用程序服务器提供了几个[Non-functional requirement](https://en.wikipedia.org/wiki/Non-functional_requirement#Examples)，例如安全性，隔离性，容错性，配置管理等。 做个类比，应用程序好比CD，应用程序服务器就好比是CD播放器。

作为开发人员，您将负责遵循预定义的标准，并以特定格式分发应用程序，而另一方面，应用程序服务器将“执行”您的应用程序，并提供可能因不同“品牌”而异的其他功能。 在Java世界中，应用程序服务器提供的企业功能标准最近已在Eclipse基础之下有了新的发展。 基于Eclipse Enterprise for Java（EE4J[Eclipse Enterprise for Java](https://projects.eclipse.org/projects/ee4j)）的工作已经诞生了[Jakarta EE](https://jakarta.ee/)。 （欲了解更多信息，请阅读文章[Jakarta EE正式发布](https://developers.redhat.com/blog/2018/04/24/jakarta-ee-is-officially-out/)或观看DevNation视频[Jakarta EE：Java EE的未来](https://developers.redhat.com/videos/youtube/f2EwhTUmeOI/)。）

继续CD播放器的类比，随着容器的提升，容器镜像已经成为新的CD格式。 事实上，容器镜像只不过是分发容器的格式。（如果您需要更好地处理容器镜像以及它们如何分发，请参阅[容器术语的实用介绍](https://developers.redhat.com/blog/2018/02/22/container-terminology-practical-introduction/)。）

当您需要向应用程序添加企业级功能时，才能体会到容器的真正好处。 向容器化应用程序提供这些功能的最佳方式是使用Kubernetes作为它们的平台。 此外，Kubernetes平台为其他项目（如Red Hat OpenShift，Istio和Apache OpenWhisk）提供了良好的基础，可以构建和部署强大的生产级别质量的应用程序。

让我们来探索其中的九个功能：

![Screenshot-2018-05-18-21.20.31.png](https://developers.redhat.com/blog/wp-content/uploads/2018/05/Screenshot-2018-05-18-21.20.31.png)

### 1-服务发现
服务发现是确定如何连接到某个服务的过程。 要获得容器和云原生应用程序的诸多好处，您需要从容器镜像中隔离配置，以便在所有环境中使用相同的容器镜像。 应用程序的外部化配置是`12-factor`应用程序的关键原则之一。 服务发现是从运行时环境获取配置信息的方法之一，而不是在应用程序中进行硬编码。 Kubernetes提供开箱即用的服务发现。 Kubernetes还提供ConfigMaps和Secrets以从应用程序容器中隔离配置。 Secerets解决了当您需要存储凭据以连接到运行时环境中的数据库等服务时出现的一些挑战。

使用Kubernetes，不需要使用外部服务器或框架。 尽管您可以通过Kubernetes YAML文件管理每个运行时环境的环境设置，但红帽OpenShift提供了一个GUI和CLI，可以让DevOps团队更容易管理。

### 2-基本调用

在容器内部运行的应用程序可以通过Ingress访问进行访问 - 换句话说，就是从外部世界到您正在暴露的服务。 OpenShift使用HAProxy提供路由对象，HAProxy具有多种功能和负载均衡策略。 您可以使用路由功能进行滚动部署。 这是一些非常复杂的CI / CD策略的基础。 请参阅下面的“6 - 构建和部署管道”。

如果您需要运行一次性作业（例如批处理），或者只是利用群集来计算结果（例如计算Pi的小数位），该怎么办？ Kubernetes为这个用例提供了job对象。 还可以使用cron job来管理基于时间的job。

### 3-伸缩性

Kubernetes通过使用ReplicaSets（以前称为Replication Controllers）来解决弹性问题。 就像Kubernetes的大多数配置一样，ReplicaSet是协调所需状态的一种方式：您告诉Kubernetes系统应该处于什么状态，Kubernetes会如何实现它。 ReplicaSet可以随时控制应运行的应用程序的副本数量或确切副本数量。

但是，如果您构建的服务比您计划的更受欢迎或者计算资源耗尽，会发生什么？ 您可以使用Kubernetes Horizontal Pod Autoscaler，它根据观察到的CPU利用率（或者通过自定义度量标准支持，或者根据某些其他应用程序提供的度量标准）来自动伸缩Pod数量。

### 4-日志

由于您的Kubernetes集群运行着容器化应用程序的多个副本，因此集中这些日志以便可以在一个位置查看它们变得非常重要。 此外，为了利用自动缩放（以及其他云原生功能）等优势，您的容器必须是不可变的。 所以你需要将你的日志存储在你的容器之外，这样它们将在整个运行期间保持不变。 OpenShift允许您部署EFK堆栈以聚合来自主机和应用程序的日志，无论它们来自多个容器，还是来自已经删除的容器。

EFK堆栈由以下部分组成：
* Elasticsearch（ES），一个存储所有日志的对象存储系统
* Fluentd，从节点收集日志并将其提供给Elasticsearch
* Kibana，Elasticsearch的Web用户界面

### 5-监控

虽然日志和监控看起来是为了解决同样的问题，但它们彼此不同。监控是观察，检查，异常报警，以及记录。 而日志仅仅是记录。

Prometheus是一个开源的监控系统，包含时序数据库。 它可用于存储和查询度量标准，提醒并使用可视化来深入了解您的系统。 Prometheus可能是监控Kubernetes集群最流行的选择。 在Red Hat Developers博客上，有几篇文章涉及如何使用Prometheus进行监控。 您还可以在OpenShift博客上找到Prometheus相关的文章。

您也可以通过https://learn.openshift.com/servicemesh/3-monitoring-tracing与Istio一起参与Prometheus。


### 6-编译和部署管道

CI / CD（持续集成/持续交付）管道不是严格的“必备要求”。 但是，CI / CD通常被视为成功的软件开发和DevOps实践的支柱。 所有软件都应该通过CI / CD管道部署到生产环境中。 Jez Humble和David Farley撰写的[持续交付：通过构建，测试和部署自动化实现可靠的软件发布](https://www.amazon.com/dp/0321601912?tag=contindelive-20)一书中如此描述CD：“持续交付是能够获取所有类型的更改 - 包括新功能，配置更改，错误 修复和实验 - 以可持续的方式安全快速地投入到生产或用户手中。“

OpenShift将CI / CD流水线作为“构建策略”且开箱即用。请查看我两年前录制的一个[视频](https://www.youtube.com/watch?v=N8R3-eNVoEc)，一个新微服务的Jenkins CI / CD管道的示例。

### 7-弹性

虽然Kubernetes为群集本身提供了弹性选项，但它还可以通过提供支持replicated volumes的PersistentVolumes来帮助应用程序变得更加具有弹性。 Kubernetes的ReplicationController / deployments可确保指定数量的pod副本在整个群集中一致地部署，从而自动处理任何可能的节点故障。

结合弹性，容错功能是解决用户可靠性和可用性问题的有效手段。 通过它的重试规则，断路器和pool ejection，也可以通过Istio为运行在Kubernetes上的应用程序提供容错功能。 请参阅 https://learn.openshift.com/servicemesh/7-circuit-breaker 上的Istio Circuit Breaker教程。

### 8-鉴权

Istio还可以通过其相互TLS身份验证来提供Kubernetes中的身份验证，该身份验证旨在增强微服务及其通信的安全性，而无需更改服务代码。 它负责：

* 为每个服务提供基于角色的身份，以实现跨群集和云的互操作性
* 确保服务到服务的通信和终端用户到服务的通信的安全性
* 提供密钥管理系统，自动化密钥和证书生成，分发，轮换和撤销

此外，值得一提的是，您也可以在Kubernetes / OpenShift群集中运行Keycloak以提供身份验证和授权。 Keycloak是Red Hat Single Sign-on的上游产品。 有关更多信息，请阅读使用[Keycloak轻松实现单点登录](https://developers.redhat.com/blog/tag/keycloak/)。 如果您使用的是Spring Boot，请观看DevNation视频：[使用Keycloak安全部署Spring Boot Microservices](https://developers.redhat.com/videos/youtube/Bdg_DjuoX0A/)或阅读[博客文章](https://developers.redhat.com/blog/tag/keycloak/)。

### 9-链式追踪

启用Istio的应用程序可以使用Zipkin或Jaeger实现分布式追踪。 无论您使用何种语言，框架或平台构建应用程序，Istio都可以启用分布式跟踪。 请访问 https://learn.openshift.com/servicemesh/3-monitoring-tracing 查看详情。 另请参阅[Laptop上的Istio和Jaeger入门](https://developers.redhat.com/blog/2018/05/08/getting-started-with-istio-and-jaeger-on-your-laptop/)以及最近的DevNation视频：[使用Jaeger跟踪高级微服务](https://developers.redhat.com/blog/2018/06/20/next-devnation-live-advanced-microservices-tracing-with-jaeger-june-21st-12pm-edt/)。


## 应用程序服务器已经成为明日黄花？

通过这些功能，您可以了解Kubernetes + OpenShift + Istio如何真正为您的应用程序提供支持，并提供过去由应用程序服务器或Netflix OSS等软件框架负责的功能。 这是否意味着应用服务器已经死亡？

在新的容器化世界里，应用服务器正在变得越来越像框架。软件开发的发展很自然地导致了应用服务器的发展。 这种演变的一个很好的例子是Eclipse MicroProfile规范，它将WildFly Swarm作为应用程序服务器，为开发人员提供容错，配置，跟踪，REST（客户端和服务器）等功能。 但是，WildFly Swarm和MicroProfile规范设计得非常轻巧。 WildFly Swarm没有完整的Java企业应用程序服务器所需的大量组件。 相反，它专注于微服务，并拥有足够的应用程序服务器，来构建和运行您的应用程序，就像一个可执行.jar文件那么简单。 您可以在博客上阅读有关[MicroProfile](https://developers.redhat.com/blog/tag/microprofile/)的更多信息。

此外，Java应用程序可以具有诸如Servlet引擎，数据源池，依赖注入，事务，消息传递等功能。 当然，框架可以提供这些功能，但是应用程序服务器还必须拥有在任何环境中构建，运行，部署和管理企业应用程序所需的一切，无论它们是否在容器内。 事实上，应用服务器可以在任何地方执行，例如，裸机，虚拟化平台（如Red Hat Virtualization），私有云环境（如Red Hat OpenStack Platform），以及公共云环境（如Microsoft Azure或Amazon Web Service)。

一个好的应用程序服务器可以确保提供的API与其实现之间的一致性。 开发人员可以确定部署其业务逻辑（需要的各种功能）将正常工作，因为应用程序服务器开发人员（以及定义的标准）已确保这些组件协同工作并一起发展。 此外，一个好的应用服务器还负责最大化吞吐量和可扩展性，因为它将处理来自用户的所有请求; 减少延迟并缩短加载时间，因为它有助于您的应用程序的可处置性; 轻量级，内存占用少，可最大限度地减少硬件资源和成本; 最后，要足够安全以避免任何安全漏洞。 对于Java开发人员，Red Hat提供了红帽JBoss企业应用程序平台，它满足了现代模块化应用程序服务器的所有要求。

## 结论

容器镜像已成为分布式云原生应用程序的标准打包格式。 虽然容器“本身”不能为应用程序提供真正的业务优势，但Kubernetes及其相关项目（如OpenShift和Istio）提供了以前属于应用程序服务器的非功能性需求。

开发人员过去从应用程序服务器或诸如Netflix OSS之类的库中获得的大多数非功能性需求都绑定到特定语言，例如Java。 另一方面，当开发人员选择使用Kubernetes + OpenShift + Istio满足这些要求时，他们不会附加到任何特定语言，这可以鼓励为每个用例使用最佳技术/语言。

最后，应用服务器仍然在软件开发中占有一席之地。 但是，它们正在变得更像是特定于语言的框架，这是开发应用程序时的一个很好的捷径，因为它们包含许多已经编写和测试过的功能。

迁移到容器，Kubernetes和微服务的最好的事情之一就是，您不必为您的应用程序选择单个应用程序服务器，框架，架构风格甚至语言。 您可以使用运行现有Java EE应用程序的JBoss EAP轻松部署容器，也可以使用Wildfly Swarm或Eclipse Vert.x进行响应式编程的新微服务。 这些容器都可以通过Kubernetes进行管理。 要了解这个概念，请查看[Red Hat OpenShift Application Runtimes](https://developers.redhat.com/products/rhoar/overview/)。 讲述了如何利用[Launcher Service](https://developers.redhat.com/launch/)来在线构建和部署一个使用WildFly Swarm，Vert.x，Spring Boot或Node.js的示例应用程序。 还可以通过选择Externalized Configuration任务以了解如何使用Kubernetes ConfigMaps。 这将是您开始学习云原生应用程序的良好开端。

你可以说Kubernetes/OpenShift是新的Linux，也可以说Kubernetes是新的应用程序服务器。但事实是应用程序服务器/运行时+OpenShift/ Kubernetes + Istio已成为“事实上的”云原生应用程序平台！

如果您最近没有访问过Red Hat Developer网站，请查看以下页面：

* [Kubernetes和容器管理](https://developers.redhat.com/topics/kubernetes/)
* [微服务指南](https://developers.redhat.com/topics/microservices/)
* [服务网格和Istio指南](https://developers.redhat.com/topics/service-mesh/)

#### 关于作者

Rafael Benevides是Red Hat的开发体验总监。 凭借在IT行业多个领域的多年经验，他帮助世界各地的开发人员和公司更有效地进行软件开发。 Rafael认为自己是一个对分享有着浓厚兴趣的问题解决者。 他是Apache DeltaSpike PMC(一个Duke's Choice Award奖得主项目)的成员以及JavaOne，Devoxx，TDC，DevNexus等许多其他会议的发言人。