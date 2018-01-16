---
title: helm -- kubernets 的包管理工具
date: 2018-01-16 20:21:58
categories: k8s
tags: [helm]
---

Helm 是 Kubernetes 的一个包管理工具，用来简化 Kubernetes 应用的部署和管理。可以把 Helm 比作 CentOS 的 yum 工具， 而我们的服务可以看作是 rpm 包。

<!-- more -->

### 1. 基本概念
1）Chart: 是 Helm 管理的安装包，里面包含需要部署的安装包资源<br>
2）Release：是 chart 的部署实例，一个 chart 在一个 Kubernetes 集群上可以有多个 release，即这个 chart 可以被安装多次<br>
3）Repository：chart 的仓库，用于发布和存储 chart<br>

### 2. 组成
Helm 是 C/S 架构， 有 client 和 server 构成。<br>
`tiller`(server): 运行在 Kubernetes 集群上;<br>
`helm`(client,命令行工具)：可在本地运行，一般运行在CI/CD Server上

### 3. 安装 和 初始化
安装教程， 以及初始化教程， 可以参考 [helm github install.md](https://github.com/kubernetes/helm/blob/master/docs/install.md)

### 4. role-based access control(基于角色的访问控制)
在 Kubernetes 1.8，Kubernetes APIServer 正式开启了RBAC访问控制，所以我们需要创建 tiller 使用的service account: tiller 并分配合适的角色给它。本文不多做说明， 关于如何配置请参考[tiller and RBAC](https://docs.helm.sh/using_helm/#role-based-access-control)。至于 RBAC， 可以参考 [Hands on with RBAC in Kubernetes 1.8](https://coreos.com/blog/hands-on-with-rbac-in-kubernetes-1.8)

### 5. 基本使用
#### 5.1 创建 chart
```
$ helm tree hello-world 
hello-world
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── ingress.yaml
│   ├── NOTES.txt
│   └── service.yaml
└── values.yaml

2 directories, 7 files
```
<ol>
<li>charts: 本chart依赖的chart，当前是空的;</li>
<li>Chart.yaml: 基本信息


```
apiVersion: v1
description: A Helm chart for Kubernetes
name: microservice
version: 0.1.0
```

</li>
<li>templates: 资源定义文件（serverce， deployment ..etc）</li>
<li>values.yaml: chart 配置默认值， 在 templates 文件中调用</li>
</ol>

#### 5.2 安装
直接在上文创建出来的 `hello-world` chat 目录下， 执行<br>
`helm install ./`
查看 release：<br>
`helm list`
删除 release： <br>
`helm delete release-name`
#### 5.3 打包分发
```
$ helm package ./
Successfully packaged chart and saved it to: /home/work/kube/helm/hello-world/hello-world-0.1.0.tgz
```
同样可以根据此 tgz 包来进行安装、回滚、升级
```
$ helm install hello-world-0.1.0.tgz
$ helm rollback releasename 1 # releasename 由 helm list 获得， 回滚前一个版本
$ helm upgrade releasename .  # 更新过 chart 并且 package 后， upgrade 替代 install 命令， 进行更新 
```

参考：<br>
[是时候使用Helm了：Helm, Kubernetes的包管理工具](https://www.kubernetes.org.cn/3435.html)<br>
[TILLER AND ROLE-BASED ACCESS CONTROL](https://docs.helm.sh/using_helm/#role-based-access-control)<br>
[helm github docs](https://github.com/kubernetes/helm/tree/master/docs)<br>
[Using Helm to deploy to Kubernetes](https://daemonza.github.io/2017/02/20/using-helm-to-deploy-to-kubernetes/)<br>






