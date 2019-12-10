---
layout: post
title: Kubernetes(3) 实战
category: 技术
tags: kubernetes
keywords: kubernetes
---

## 前言

最近因为工作上的一些原因，在搭Kubernetes集群，发现我以前看的书真是太老了。。。很多概念都已经不适用了，并且有了很多替代方案，比较好的学习方式还是看官方文档啊_(′ཀ｀」∠)_，唉。

下面以redis-ha集群为例，记录一下使用心得

## 存储

### PersistentVolume

简单来说，一个PV对应一段存储，这段存储的实际形态可以是不同的（如：NFS、本地磁盘或者Azure Filesystem之类的云存储），这些具体存储被抽象成PV。并且，它的生命周期是独立于Pod的，我们可以另外选择是否保留、回收或删除。

### PersistentVolumeClaim

可以说PVC位于PV的上一层，Pod通过PVC请求特定的资源（可以指定大小和访问模式），PVC根据请求，绑定符合条件的PV资源，以供Pod使用。PVC和PV一一对应。

### StorageClass

存储类类似一种描述，可以动态分配PV资源，并且指定PV的回收策略。

### Example

首先我们创建一个名为redis-sc的存储类

```
# redis-sc.yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
    name: redis-sc
provisioner: Local
reclaimPolicy: Retain
volumeBindingMode: Immediate
```

然后创建3个PV，通过``spec.storageClassName``将其分配给上面我们创建好的存储类。
这里我选择的PV类型是NFS，挂载到开发机上，这样的好处是，我在任何机器上启动的服务，最终都会落在开发机上，之后扩展为多Kubernetes集群，也有一个比较好维护的DC。

```
apiVersion: v1
kind: PersistentVolume
metadata:
    name: redis-0
spec:
    capacity:
      storage: 10Gi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle
    storageClassName: redis-sc
    nfs:
      path: /data00/kubernetes/redis0
      server: ${MY_DEV_HOST}"
```

## 开始

### ConfigMap

```
# redis.conf
appendonly yes
cluster-enabled yes
cluster-config-file /var/lib/redis/nodes.conf
cluster-node-timeout 5000
dir /var/lib/redis
port 6379
```

### Headless Service

Headless Service是一种特殊的服务，特征在于``spec.clusterIP: None``，这里用到它是因为，因为无头服务可以给每个Pod一个唯一的名称，这样在Pod重新调度时，它的名称可以保证不变，StatefulSet可以通过无头服务实现稳定的网络标识。

```
# redis-service.yml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  labels:
    app: redis
spec:
  ports:
  - name: redis-port
    port: 6379
  clusterIP: None
  selector:
    app: redis
    appCluster: redis-cluster
```

### StatefulSet

我上次看书的时候，还只有Deployment这一种服务部署的概念，现在已经有很多了（啊，青春！）

很明显嘛，StatefulSet就是管理有状态服务的集合，特点就是Pod具有固定的名称和启停顺序，其中特别解释一下``volumeClaimTemplates``这个概念：

分布式系统的数据是不一样的，因此Pod间不能共用存储卷，每个Pod要有自己专有的存储卷，所以StatefulSet使用这个Template为每个Pod生成不同的PVC，绑定不同的PV。这也是我们在最开始讲存储的时候没有创建PVC的原因。

下面的文件好长，但我还是贴了，哎。
```
# redis-statefulset.yml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: redis-app
spec:
  serviceName: "redis-service"
  replicas: 3
  template:
    metadata:
      labels:
        app: redis
        appCluster: redis-cluster
    spec:
      terminationGracePeriodSeconds: 20
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - redis
              topologyKey: kubernetes.io/hostname
      containers:
      - name: redis
        image: redis:3.2.8
        command:
          - "redis-server"
        args:
          - "/etc/redis/redis.conf"
          - "--protected-mode"
          - "no"
        resources:
          requests:
            cpu: "100m"
            memory: "100Mi"
        ports:
            - name: redis
              containerPort: 6379
              protocol: "TCP"
            - name: cluster
              containerPort: 16379
              protocol: "TCP"
        volumeMounts:
          - name: "redis-conf"
            mountPath: "/etc/redis"
          - name: "redis-data"
            mountPath: "/var/lib/redis"
      volumes:
      - name: "redis-conf"
        configMap:
          name: "redis-conf"
          items:
            - key: "redis.conf"
              path: "redis.conf"
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
      storageClassName: redis-sc
      volumeMode: Filesystem

```

## 通过Ubuntu容器管理Redis

这里我们通过基础容器来使用[redis-trib](http://weizijun.cn/2016/01/08/redis%20cluster%E7%AE%A1%E7%90%86%E5%B7%A5%E5%85%B7redis-trib-rb%E8%AF%A6%E8%A7%A3/)这个工具

```
$ kubectl run -i --tty ubuntu --image=ubuntu --restart=Never /bin/bash
```

进入容器进行环境安装：
1. 换源
2. 安装redis-trib

```
$ apt-get update
$ apt-get install -y vim wget python2.7 python-pip redis-tools dnsutils
$ pip install redis-trib
```

3. 通过redis-trib添加master节点

```
$ redis-trib.py create `dig +short redis-app-0.redis-service.default.svc.cluster.local`:6379
```

4. 为master节点添加slave

```
$ redis-trib.py replicate \
  --master-addr `dig +short redis-app-0.redis-service.default.svc.cluster.local`:6379 \
  --slave-addr `dig +short redis-app-1.redis-service.default.svc.cluster.local`:6379

$ redis-trib.py replicate \
  --master-addr `dig +short redis-app-0.redis-service.default.svc.cluster.local`:6379 \
  --slave-addr `dig +short redis-app-2.redis-service.default.svc.cluster.local`:6379
```
注：该容器后续可清除

## 后续

### Access Service

按照道理来说，到了这一步，我们的集群已经搭建好了，不过因为我要把端口暴露到宿主机，所以ClusterIP模式下的Headless Service不能满足我的需求，所以我还需要创建一个NodePort模式的服务提供外部访问：

```
# redis-access-service.yml
apiVersion: v1
kind: Service
metadata:
  name: redis-access-service
  labels:
    app: redis
spec:
  ports:
  - name: redis-port
    protocol: "TCP"
    port: 6379
    targetPort: 6379
    nodePort: 30010
  selector:
    app: redis
    appCluster: redis-cluster
  type: NodePort
```

### NodeID

这里再可以扩展一个知识点，就是其实有了Access Service之后，前面的Headless Service就不需要了。为什么呢？因为在Redis集群中，每个节点都有自己的NodeID，保存在配置文件里，IP变化时，NodeID并不会随之变化，这也是一种固定的网络标志。

NodeID的两种使用场景：

> * 当某个Slave Pod断线重连后IP改变，但是Master发现其NodeId依旧， 就认为该Slave还是之前的Slave。
> * 当某个Master Pod下线后，集群在其Slave中选举重新的Master。待旧Master上线后，集群发现其NodeId依旧，会让旧Master变成新Master的slave。


## 后后后续

为什么我消失了2个月！！！因为我最近工作好忙呜呜呜呜，根本没空上班学习造轮子看paper（🦑），好在最近的工作事项有一部分是要搭建一个完整封闭的k8s集群，感觉还是学到了很多新知识。要不然我已经是个废物业务仔了。

马上2020年了，我的新目标还没定， (ง •̀_•́)ง冲冲冲！