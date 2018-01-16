---
title: kubernetes入门
date: 2018-01-15 19:40:26
categories: k8s
tags: [k8s]
---

首先， Kubernetes(k8s)是一个全新的基于容器(docker/Rocket)技术的分布式架构解决方案。其次，是一个开放的开发平台，不同的是没有限定任何变成接口、语言、框架，可以很简单的将系统映射为k8s的Service。最后， 是一个一站式的完备的分布式系统支撑平台，k8s具有完备的集群管理能力，包括多层次的安全防护和准入机制、多租户的应用支撑能力、透明的服务注册和发现机制、内建智能负载均衡器、故障发现和自我修复能力、服务滚动升级和在线扩容能力、可扩展的资源自动调度机制，以及多粒度的资源配额管理能力。同时提供了涵盖开发、部署测试、运维监控在内的各个环节的管理工具。
<!-- more -->


### 1. Master(主节点)
组成： <br>
1）APIServer： 负责对外提供 RESTful 的 k8s API服务，它是系统管理指令的统一入口，任何对资源进行增删改查的操作都要交给APIServer处理后再提交给etcd<br>
2）scheduler： 负责调度pod到合适的Node上<br>
3）Controller manager： 负责管理资源控制器（每个资源一般都对应有一个控制器）<br>
4）etcd： 存储各个资源的状态，从而实现了Restful的API<br>

### 2. Node(节点)
组成：<br>
1）Runtime： 指的是容器运行环境<br>
2）kube-proxy： 服务发现和反向代理功能<br>
3）kubelet： 是Master在每个Node节点上面的agent， 负责维护和管理该Node上面的由k8s创建的所有容器<br>

创建过程：<br>
1）创建： k8s在物理机、虚拟机或其他云服务资源上创建一个 node 对象， 并对其进行一系列健康检查（是否可以联通、服务是否正确启动、是否可以创建pod等），然后在集群中标记其状态。<br>
1）管理： k8s master 通过 Node Controller 管理集群内 node 的信息同步、生命周期等；<br>
1）注册: k8s 推荐自注册， 由 kubelet 向 Apiserver 注册自己， 也可以手动注册。<br>

### 3. Pod 
在 k8s 中， Pod 是其基本操作单元，也是应用运行的载体。Pod 包含一个或者多个相关的容器（**可以看成应用层的逻辑宿主机**）。Pod 可以认为是容器的一种延伸扩展，一个Pod也是一个隔离体，而Pod内部包含的一组容器又是共享的。除此之外，Pod中的容器可以访问共同的数据卷来实现文件系统的共享。
> Pod 同 docker 一样， 通过数据卷挂载来实现数据持久化（本地、网络等数据卷）。<br>
Pod 封装 docker 的目的： docker 容器通信受 docker 网络机制限制（--link），通过 Pod 概念将容器组合在一个虚拟主机内，实现容器通过 localhost 通信（共享 PID 命名空间，网络命名空间，主机名，存储卷等，实现进程通信等）。<br>

Pod 定义： 通过Yaml或Json格式的配置文件来完成, 示例如下：<br>
```
apiVersion: v1
kind: Pod
metadata:
    name: redis-slave
    labels:
        name: redis-slave
spec:
    containers:
    - name: slave
      image: kubeguide/guestbook-redis-slave
      command： ['sh']
      env:
    - name: GET_HOSTS_FROM
      value: env
      ports:
```
Pod 基本操作： <br>
增：`kubectl create -f xxx.yaml`<br>
删： `kubectl delete pod yourPodName` <br>
改： `kubectl replace /path/to/yourNewYaml.yaml`<br>
查： `kubectl get pod yourPodName`， `kubectl describe pod yourPodName` 

### 4. Label
Label定义如 Pod、Service、RC、Node 等对象的可识别属性（key/value 键值对），用来对它们进行管理和选择，可以如上面示例那样定义以供创建时附加到对象，也可以在对象创建后通过API进行管理。通过 `selector` 来进行 label 选择，进行资源之间的关联。<br>
如下代码所示， 将该资源与 `name： redis-slave` 的资源进行关联。
```
selector:
  name: redis-slave
```

### 5. Replication Controller（RC）
用于定义 Pod 副本的数量， 在Master内，Controller Manager进程通过RC的定义来完成Pod的创建、监控、启停等操作。多停少启，保证集群中运行着用户期望的副本数量。<br>
与 Pod 类似， 通过 yaml 或 json 配置文件来定义。示例如下， 相比 Pod 定义， `kind` 变为 ReplicationController， 另外多了 `replicas`， 以及 Label 选择器 `selector`。 同时在 `template` 中定义了 Pod， 由于 RC 特性，通常通过这种形式来创建资源。
```
apiVersion: v1
kind: ReplicationController
metadata:
  name: redis-slave
  labels:
    name: redis-slave
spec:
  replicas: 3
  selector:
    name: redis-slave
  template: # template 内定义 Pod， 
    metadata:
      name: redis-slave
      labels:
        name: redis-slave
    spec:
      containers:
      - name: slave
        image: kubeguide/guestbook-redis-slave
        command： ['sh']
        env:
      - name: GET_HOSTS_FROM
        value: env
        ports:
```

### 6. Deployment
> 后续发展中，出现了 Replica Set – 下一代Replication Controller。相比与 RC `selector` 的 key/value， RS 还支持集合操作（in,notin ..etc）,后续肯定会加入更多功能.
Deployment使用了Replica Set，是更高一层的概念。同时，与 RC 相比， Deployment 拥有更加灵活强大的升级、回滚功能。除非需要自定义升级功能或根本不需要升级 Pod，所以推荐使用 Deployment 而不直接使用Replica Set.<br>
Deployment 的定义 与 RC 类似， 主要变化：
```
apiVersion: extensions/v1beta1
kind: Deployment
```

### 7. Service（服务）
Service 是一种抽象概念，它定义了一个 Pod 逻辑集合以及访问它们的策略。Service 可以看作一组提供相同服务的 Pod 的对外访问接口， 即 Pod 组成的集群。在受到 RC 调控的时候，Pod副本是变化的，对应的虚拟 IP 也是变化的，但 Service 的 Cluster IP 地址是 Kubernetes 系统中的虚拟IP地址(虚拟IP的范围通过k8s API Server的启动参数 --service-cluster-ip-range=19.254.0.0/16配置)，由系统动态分配，在销毁该 Service 之前，这个 IP 地址都不会再变化了。
这样， Service就可以作为 Pod 的访问入口，起到代理服务器的作用（同时可以代理 k8s 外部的服务），而对于访问者来说，通过Service进行访问，无需直接感知 Pod。<br>
>虚拟 IP 属于 k8s 内部的虚拟网络，外部是寻址不到的。在 k8s 系统中，实际上是由 k8s Proxy组件负责实现虚拟 IP 路由和转发的，所以k8s Node中都必须运行了k8s Proxy，从而在容器覆盖网络之上又实现了k8s层级的虚拟转发网络。<br>
>将服务发布至外部网络访问[方式](http://alesnosek.com/blog/2017/02/14/accessing-kubernetes-pods-from-outside-of-the-cluster/)<br>

由于 pod 会随时销毁重建， 所以 k8s 会根据 Service 关联到 Pod 的 PodIP 信息组合成一个 Endpoints，用来相互衔接， 并且 Endpoints 会随着 pod 的变化而变化

### 8. Volume（存储卷）
k8s 的 Volume 概念与 Docker 的 Volume 比较类似，但不完全相同， 有`本地`、`网络`、`信息`等几种类型的数据卷。

### 9. 简单创建资源流程
假设有 `Deployment` 定义文件 `deployment-demo.yaml` 和 `Service` 定义文件 `service-demo.yaml`
```
$ kubectl create deployment-demo.yaml # 创建控制器和 Pod
$ kubectl create service-demo.yaml    # 创建 service 和 endpoints
$ kubectl get all                     # 查看所有资源信息
```

> 参考： <br>
[kubernetes入门](https://www.jianshu.com/p/63ffc2214788)<br>
[Kubernetes核心概念总结](https://www.cnblogs.com/zhenyuyaodidiao/p/6500720.html)<br>
[k8s 概念](https://kubernetes.io/docs/concepts/)<br>
[Accessing Kubernetes Pods from Outside of the Cluster](http://alesnosek.com/blog/2017/02/14/accessing-kubernetes-pods-from-outside-of-the-cluster/)



 



