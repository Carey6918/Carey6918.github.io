---
layout: post
title: Kubernetes(2) 基础概念
category: 技术
tags: kubernetes
keywords: kubernetes
---

## Master
集群控制节点；控制命令都由它来执行；它占据一个独立的X86服务器（或虚拟机），因为它非常重要，一旦宕机，会导致所有控制命令失效

![Master](https://raw.githubusercontent.com/Carey6918/Carey6918.github.io/master/assets/images/kubernetes-2-1.png)

### Master中的进程

- kube-apiserver，提供HTTP Rest接口； 是增删改查k8s中资源的入口，也是集群控制的入口。
- kube-controller-manager，资源对象的自动化控制中心
- kube-scheduler，负责资源调度（Pod调度）
- etcd-server，将资源对象的数据保存在etcd中


## Node

集群中的其他机器；每个Node被Master分配一些工作负载（Docker容器），当某个Node宕机时，工作负载会被Master转移到其他Node

![Node](https://raw.githubusercontent.com/Carey6918/Carey6918.github.io/master/assets/images/kubernetes-2-2.png)

### Node中的进程
- kubelet，负责Pod对应的容器的创建、启停等任务，与Master协作，实现集群管理的基本功能。
- kube-proxy，实现k8s服务的通信与负载均衡
- docker：负责容器创建和管理

### Node如何工作
1. 一个节点正确安装、配置和启动了上述进程
2. 该节点被增加到k8s集群中
3. kubelet进程定期向Master发送心跳，汇报自身情况，例如os、version、cpu、内存、pod等情况
4. Master获得信息后，可以合理分配资源进行负载均衡
5. 超过一定时间没有心跳后，Master认为该Node失联，触发Docker转移

```
// 查看集群中有多少个Node

$ kubectl get nodes

// 查看某个Node的具体信息

$ kubectl describe node node-name
```

## Pod
Pod代表集群中运行的进程，是一个基本单位
每个Pod都有一个“Pause”（根容器），Pause容器对应的镜像属于k8s平台的一部分；
每个Pod还包含一个或多个用户业务容器

### Why Pod?
1. 管理：引入业务无关且不易死亡的Pause容器作为根容器，以它的状态作为整个容器组的状态，更好的抽象。
2. 资源共享和通讯：每个Pod都有自己的IP，彼此之间互相联通。

### Pod如何启动

#### 静态Pod
存放在具体某个Node的一个具体文件中，只在此Node上启动运行
#### 普通Pod
1. 创建一个Pod，并存储到etcd中
2. 被k8s Master调度到某个Node上，并绑定
3. 该Node的kubelet进程将这个Pod实例化为一组容器并启动起来
4. 当Pod中的某个容器挂掉时，k8s会重启这个Pod中的所有容器
5. 当Pod所在Node挂掉时，k8s会将这个Node上的所有Pod重新调度到其他Node上

## Label
一个Label是一个kv键值对，可以附加到任何资源上，例如Node、Pod、Service、RC等
我们可以给制定的资源对象绑定一个或多个不同的Label来实现多维度资源分组管理。
### 常用Label
- 版本：“release":"stable"; "release":"canary" ...
- 环境："environment":"dev"; "environment":"qa"; "environment":"production"...
- 架构："tier":"backend"; "tier":"frontend"; tier:"middleware"
- 分区："partition":"customerA"; ...
- 质量管控："track":"daily"; "track":"weekly"

### 使用场景
- kube-controller通过RC的Label来筛选要监控的Pod副本数量，从而实现Pod副本数量始终符合预期的自动控制流程。
- kube-proxy通过Service的Label来选择Pod，自动简历每个Service到对应Pod的路由表，实现Service的智能LB机制。
- 对某些Node定义特定Label，并在Pod定义文件中指定Node Label，可以通过kube-scheduler实现Pod定向调度

## Replication Controller(RC)

### 定义
- Pod期待的副本数（Replica）
- 用于筛选目标Pod的Label Selector
- 创建新Pod的模板

### RC如何工作
1. 创建一个RC
2. kube-controller-manager得到通知，定期巡检集群中存活的目标Pod，确保目标Pod的实例数量刚好等于RC期望值，如果过多，就杀死一些Pod，如果过少，就创建一些Pod
3. RC保证了集群中始终有且仅有n个目标Pod在运行，并且可以实现动态缩放
4. 删除RC时不会删除该RC对应的Pod，可以将replicas设为0

### 滚动升级
旧版本的Pod每次停止一个，同时创建一个新的Pod，但是集群中Pod数量保持不变

## Deployment
可以看作是RC的一次升级，我们可以随时知道当前Pod部署的进度。

### Deployment如何工作
1. 检查一个Deployment对象来生成对应的Replica Set（支持集合的Label Selector，升级版RC），并完成Pod的创建过程
2. 检查Deployment的状态来看部署是否完成（检查Pod数量是否达到预期值）
3. 更新Deployment来创建新的Pod
4. 如果当前Deployment不稳定，回滚到上一个版本
5. 挂起或者恢复一个Deployment

```
// 创建Deployment:
$ kubectl create -f example-deployment.yaml

// 查看Deployment信息
$ kubectl get deployments
> NAME          DESIRED          CURRENT        UP-TO-DATE      AVAILABLE      AGE
> example       1                1              1               1              3m
// DESIRED：Pod副本数量的期望值，即Replica
// CURRENT：当前Replica的值，当它等于DESIRED时表示部署完成
// UP-TO-DATE：最新版本的Pod数量，用于检查有多少个Pod完成滚动升级
// AVAILABLE：当前集群中可用的Pod数量
```

## Horizontal Pod Autoscaler(HPA)
> 通过追踪分析RC控制的所有目标Pod的负载变化情况，来确定是否需要针对性地调整目标Pod的副本数，这是HPA的实现原理


### Pod负载度量指标
- 目标Pod所有副本的CPU利用率平均值
- 应用程序自定义的度量指标，比如QPS

## Service
一组Pod提供一个对外的服务端口，并将这组Pod的Endpoint列表加入服务端口的转发列表，客户端就可以通过IP+服务端口来访问此服务，并且负载均衡器转发到某个Pod

## Volume（存储卷）
是Pod中能被多个容器访问的共享目录。用途类似Docker的Volume，但不等价。
- k8s的Volume定义在Pod上，然后被一个Pod里的多个容器挂载到具体的文件目录下
- k8s的Volume生命周期与Pod相同，但与容器的生命周期不相关
- k8s支持多种类型的Volume（如GlusterFS、Ceph等）

### Volume类型
1. emptyDir：在Pod分配到Node时创建的，内容为空，自动分配；主要用于程序运行的临时目录
2. hostPath：在Pod上挂载宿主机上的文件或目录；主要用于需要永久保存的日志文件，访问Docker引擎内部的文件系统
  1. 不同的Node上的同样Pod可能会因为宿主机文件不同导致访问不一致
  2. 如果使用了资源配额管理，k8s无法管理宿主机上的资源
3. gcePersistentDisk：使用谷歌公有云的永久磁盘（PD）
  1. 限制：Node需要是GCE虚拟机；虚拟机需要与PD存在相同的GCE项目和Zone中
4. awsElasticBlockStore：使用亚马逊公有云的EBS Volume存储系统，与gce类似
5. NFS：网络文件系统......