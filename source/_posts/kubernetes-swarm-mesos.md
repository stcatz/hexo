---
title: Kubernetes对比Mesos和Docker Swarm
date: 2017-04-16 17:05:24
tags: ['Kubernetes', 'cloud', 'PaaS']
---


2016年是容器技术落地年，从技术媒体上的文章趋势来看，很多公司都在开始使用容器和容器编排系统，其中最热门的就是Kubernetes，Mesos和Docker swarm了。那么这些框架都各自有什么优缺点？本文试图从设计原则、特性、社区等几方面来对他们作一下综合比较，后续会有更多的文章详细介绍Kubernetes。

<!-- more -->

## 概述

Kubernetes缩写作K8s，脱胎于Google的Borg。由于有Google的背书，一经发布就吸引了大量眼球。K8s可以理解为一个简化的PaaS系统，提供了许多通用功能比如应用部署、扩展、日志、监控、负载均衡。开源社区有很多更完善的PaaS平台都是以K8s为核心的，比如OpenShift, Deis, Eldarion。Ubuntu也在17.04版本中加入了自己的Kubernetes发型版本[CDK](https://insights.ubuntu.com/2017/04/12/general-availability-of-kubernetes-1-6-on-ubuntu/?_ga=1.88559579.1962757385.1492152573)

Mesos是UC Berkeley RAD Lab贡献给Apache基金会的开源项目，严格意义上来说它并不是一个容器编排系统，而是一个集群管理系统。Mesosphere这家公司在2017年发布的DC/OS就是以Mesos为核心作为集群管理，加上Marathon, ZooKeeper等一其他模块一起作为一个完整的PaaS系统。所以Kubernetes应该与Mesos + Marathon作对比才有意义。

Docker已经是容器领域的事实标准，上述两种容器编排工具都默认以Docker Engine作为容器的运行时环境。而Docker作为一家创业公司显然不满足于只提供容器运行时环境，2016年把Docker Swarm整合进了Docker Engine，并推出Docker企业级版本希望抢占企业级市场。Docker Swarm所提供的编排系统比较简单，但是它也是上手最快速简单的，Docker daemon可以运行在swarm模式下，只需要简单几个命令，就可以很方便的搭建一个容器集群。


## Kubernetes

![Kubernetes architecture](http://res.cloudinary.com/dkk3prfsp/raw/upload/v1492333759/architecture_s3g9ib.png)

</p>
<center><small>图1: Kubernetes架构</small></center>


K8s的架构如图所示，节点类型可以分为master和worker。其中master节点是整个集群的控制中心，在其上运行的主要服务主要有：
* kube-apiserver: 提供API接口
* etcd: 提供分布式数据存储
* kube-controller-manager: 管理控制器
* kube-scheduler: 调度器

其他worker节点上的主要组件有：
* kubelet: 接收master节点的命令，管理本节点上的容器
* kube-proxy: 通过维护一套规则来管理网络信息
* docker/rtk: 容器运行时环境

此外还有DNS，log等服务，K8s本身很多组件也是以容器的方式部署在命名空间kube-system中。用命令`kubectl list pods —name-spaces=kube-system`就可以看到这些组件。

K8s对于容器集群中的一些通用概念做了抽象和提炼，这些名词在上手K8s的时候需要理解清楚。

* Pods: 是一个或者一组紧耦合的容器集合，也是K8s调度的最小单元。同一个Pod内的各个容器共享一个IP、volume等资源。
* Replication Controllers: 确保在一次部署中，执行监控和管理任务来确保有确定数目的冗余Pod在运行。
* Flat Networking Space: Pod内部的容器共享一个IP地址，Pods之间可以直接通信。通过修改NetworkPolicy来改变网络策略。
* Services: 一组Pods暴露出接口提供服务，可以结合Label selector来使用。
* Labels: 可以理解为tag服务，给K8s中的不同资源打上tag来方便管理。

## Mesos 和 Marathon

![DC/OS 架构](http://res.cloudinary.com/dkk3prfsp/raw/upload/v1492333814/dcos-enterprise-components-1.9_bqrgq1.png)

</p>
<center><small>图2: DC/OS架构</small></center>

Mesos可以管理的集群可以达到上万个节点。与K8s类似，Mesos的节点也可以分为master(Mesos Master)和worker(Mesos Agent)节点。Mesos提供更加底层的资源管理，每个worker提供自己的资源上报给master供调度使用。此外Mesos还使用ZooKeeper来提供分布式存储和选主服务。

Mesos要在实际应用中使用还必须提供一个框架，框架需要在Mesos Master中注册一个调度器和一个执行器来让决定具体的任务该如何调度和执行。Marathon就是作为一个框架注册在Mesos中来提供PaaS服务。Marathon也有和K8s中对应的一些概念，提供类似的功能如Pods，Services。不同点主要有：

* K8s使用go语言实现，Marathon使用了Scala。
* K8s通过对Pods, Replication Controllers, Labels对容器、服务进行管理。Mearathon抽象出树状的Appliacation Group，树之间可以有依赖关系。
* K8s通过kube-proxy和service来暴露出服务来提供HA，Marathon可以通过Mesos提供的Mesos-DNS或者Marathon-lb来提供HA。
* 网络方面，Marathon在运行Docker容器时使用host+port的方式来Map到Docker的网络上，K8s实现了重叠的网络模型，把网络区分为Pods之间的和services与其他模块的。
* Mesos在大公司的生产环境被验证过可以管理上万个节点，K8s最新的版本(1.6)声称可以支持到5000个节点。

## Docker Swarm

![Docker Swarm](http://res.cloudinary.com/dkk3prfsp/raw/upload/v1492333697/swarm-diagram_rmj2hq.png)

</p>
<center><small>图3: Doceker Swarm架构</small></center>

Docker Swarm已经整合进了Docker Engine，它的模型和提供的功能也相对比较简单。一个节点安装并启动了Docker Engine就可以调用启Swarm API让它运行在集群模式下。

```
// 创建swarm
$docker swarm init --advertise-addr <MANAGER-IP>

// 添加节点
$docker swarm join --token <token> <MANAGER-IP>
```

从架构图种可以看出，运行在Swarm模式下的Docker节点可以分为manager和worker。

* manager节点负责整个集群资源的管理和调度，并对外提供swarm API。manger节点之间使用Raft协议来同步状态。
* worker节点负责运行任务容器，manager和worker节点角色可以通过命令互换。

### 结论

在容器编排系统这个上下文中，K8s所受到的关注最多的且提供的功能较为完善。如果你希望快速搭建一套功能齐全且稳定可用的容器集群的话K8s个人认为是最佳选择。

Mesos + Marathon的优势在于高度的可定制化，其本身也经历许多公司生产环境的验证，支持多达上万台节点的部署。而且Mesos本身并不一定局限于容器的编排工具，它为上层应用提供了集群管理的基础架构，可以运行许多分布式应用，比如Spark最开始就是作为Mesos的一个数据处理框架发布的。所以如果你的集群规模庞大，而且需要运行容器以外各种类型的应用的话Mesos是最佳选择。K8s本身甚至也可以运行在Mesos之上，[这篇文章](https://mesosphere.com/blog/2015/09/25/kubernetes-and-the-dcos/)祥述了在Mesos上部署K8s的方法。

而Docker Swarm目前更适用于快速部署小规模集群，不需要额外的工具、软件，安装好Docker Engine之后简单几个命令一个容器集群就已经部署好了。

## 参考资料

* https://docs.docker.com/engine/swarm/how-swarm-mode-works
* http://mesosphere.github.io/marathon/
* http://mesos.apache.org/
* https://kubernetes.io/docs
* https://mesosphere.com/blog/2015/09/25/kubernetes-and-the-dcos/

