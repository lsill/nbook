---
title: "k8s搭建"
date: 2021-12-20T14:23:48+08:00
draft: true
---



此文为笔记，[原文](https://kuboard.cn/learning/k8s-bg/component.html#master%E7%BB%84%E4%BB%B6)

## Master 组件

Master 组件是集群的控制平台

- master 组件负责集群中的全局决策（例如：调度）
- master 组件探测并响应集群事件（例如，当Deployment的实际Pod副本数未达到replicas 字段的规定时，启动一个新的Pod）

### kube-apiserver

此 master 组件提供 Kubernetes API 。这是Kubernetes控制平台的前端，可以水平扩展（通过部署更多的实例以达到性能要求）。kubectl / Kubernetes dashboard / kuboard 等Kubernetes管理工具就是通过Kubernetes API 实现对 Kubernetes 集群的管理。



### etcd

​	支持一致性和高可用的键值对储存组件，Kubernetes集群的所有配置信息都存储在etcd中。



### kube-scheduler

​	此master 组件监控所有新创建尚未分配节点上的Pod, 并且自动选择为Pod 选择一个合适的节点去运行。

​	影响调度的因素有：

- 单个或多个Pod的资源需求
- 硬件、软件、策略的限制
- 亲和和反亲和的约定
- 数据本地化要求
- 工作负载间的相互作用



### kube-controller-manager

​	此 master 组件运行了所有的控制器。

​	从逻辑上来说，每一个控制器是一个独立的进程，但是为了降低复杂度，这些控制器都被合并运行在一个进程里。

​	kube-controller-manager 中包含的控制器有：

- 节点控制器：负责监听节点停机的事件并作出对应响应
- 副本控制器：负责为集群中每一个 副本控制器对象维护期望的Pod 副本数
- 端点控制器：负责为端点对象赋值
- Service Account & Token 控制器：负责为新的名称空间创建 default Service Account 以及 API Access Token

 

### cloud-controller-manager

cloud-controller-manager 中运行了与具体云基础设施供应商互动的控制器。

cloud-controller-manager 只运行特定于云基础设施供应商的控制器。

cloud-controller-manager 使得云供应商的代码和 Kubernetes的代码可以各自独立的演化。

以下控制器中包含与云供应商相关的依赖：

- 节点控制器：当某一个节点停止响应时，调用云供应商接口，以检查该节点的虚拟机是否已经被云供应商删除。
- 路由控制器：在云供应商的基础设施中设定网络路由。
- 服务控制器：创建、更新、删除云供应商提供的负载均衡器。
- 数据卷控制器：创建、绑定、挂载数据卷，并协调云供应商编排数据卷



### Node 组件

Node 组件运行在每一个节点上（包括master 节点和 worker 节点），负责维护运行中的Pod 并提供 Kubernetes 运行时环境。

#### Kubelet

此组件是运行在每一个集群节点上的代理程序。它确保Pod 中的容器处于运行状态。Kubelet 通过多种途径获得PodSpec 定义。并确保PodSpec 定义中所描述的容器处于运行和健康的状态。Kubelet 不管理不是通过 Kubernetes 创建的容器。



### kube-proxy

kube-proxy 是一个网络代理程序，运行在集群中的每一个节点上，是实现Kubernetes Service概念的重要部分。

kube-proxy 在节点上维护网络规划。这些网络规则使得您可以在集群内、集群外正确地与Pod进行网络通信。如果操作系统中存在packet filtering layer, kube-proxy 将使用这一特性（iptables代理模式），否则，ku-proxy将自行转发网络请求。



### 容器引擎

容器引擎负责运行容器。



### Addons

​	Addons 使用kubernetes 资源（DaemonSet、Deployment等）实现集群的功能特性。由于他们提供集群级别的功能特性，addons使用后到的Kubernetes资源都放置在kube-system 名称空间下。



### DNS

除了 DNS Addon 以外，其他的addon都不是必须的，所有Kubernetes 集群都应该有 Cluster DNS.

Cluster DNS 是一个DNS服务器，是对您已有环境中其他DNS服务器的一个补充，存放了Kubernetes Service的DNS记录。

kubernetes启动时，自动将该DNS服务器加入到容器的DNS搜索列表中。



### Web UI

Dashboard是一个Kubernetes集群的Wen管理界面。

### Kuboard

[Kuboard](https://kuboard.cn/install/v3/install.html) 是一款基于Kubernetes的微服务管理界面，相较于 Dashboard，Kuboard 强调：

- 无需手工编写 YAML 文件
- 微服务参考架构
- 上下文相关的监控
- 场景化的设计
  - 导出配置
  - 导入配置

### ContainerResource Monitoring

[Container Resource Monitoring (opens new window)](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/)将容器的度量指标（metrics）记录在时间序列数据库中，并提供了 UI 界面查看这些数据

### Cluster-level Logging

责将容器的日志存储到一个统一存储中，并提供搜索浏览的界面

