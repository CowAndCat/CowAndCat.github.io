---
layout: post
title: Kubernetes中StatefulSet介绍
category: docker
comments: false
---

StatefulSet 是Kubernetes1.9版本中稳定的特性.

# 一、StatefulSet 是什么

StatefulSet是Kubernetes提供的管理有状态应用的负载管理控制器API。在Pods管理的基础上，保证Pods的顺序和一致性。与Deployment一样，StatefulSet也是使用容器的Spec来创建Pod，与之不同StatefulSet创建的Pods在生命周期中会保持持久的标记（例如Pod Name）。

StatefulSet适用于具有以下特点的应用：

- 具有固定的网络标记（主机名）
- 具有持久化存储
- 需要按顺序部署和扩展
- 需要按顺序终止及删除
- 需要按顺序滚动更新

# 二、StatefulSet 创建
举例创建一个nginx应用，ss-nginx.yml 

    apiVersion: v1
    kind: Service
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      ports:
      - port: 80
        name: web
      clusterIP: None
      selector:
        app: nginx
    ---
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: web
    spec:
      serviceName: "nginx"
      replicas: 2
      selector:
         matchLabels:
           app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: docker.io/nginx
            ports:
            - containerPort: 80
              name: web
            volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
      volumeClaimTemplates:
      - metadata:
          name: www
        spec:
          accessModes: ["ReadWriteOnce"]
          volumeMode: Filesystem
          resources:
            requests:
              storage: 50Mi
          storageClassName: local-storage

StatefulSet创建顺序是从0到N-1，终止顺序则是相反。如果需要对StatefulSet扩容，则之前的N个Pod必须已经存在。如果要终止一个Pod，则它的后序Pod必须全部终止。

> 在Kubernetes 1.7版本后，放松了顺序的保证策略，对应的参数为 .spec.podManagementPolicy

执行创建命令，并且观察对象是否正常创建。

    [root@devops-101 ~]# kubectl create -f ss-nginx.yml 
    service "nginx" created
    statefulset "web" created
    [root@devops-101 ~]# kubectl get svc
    NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
    kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   9d
    nginx        ClusterIP   None         <none>        80/TCP    1d
    [root@devops-101 ~]# kubectl get statefulset 
    NAME      DESIRED   CURRENT   AGE
    web       2         2         1d

看一下Pod是否是有顺序的

    [root@devops-101 ~]# kubectl get pods -l app=nginx
    NAME      READY     STATUS    RESTARTS   AGE
    web-0     1/1       Running   0          11m
    web-1     1/1       Running   0          11m

根据Pod的顺序，每个Pod拥有对应的主机名，在Pod中执行hostname命令确认下。

    [root@devops-101 ~]# for i in 0 1; do kubectl exec web-$i -- sh -c 'hostname'; done
    web-0
    web-1
使用带有nslookup命令的busybox镜像启动一个Pod，检查集群内的DNS地址设置。

# 三、扩缩容
将Pod实例扩充到5个。

    [root@devops-101 ~]# kubectl scale sts web --replicas=5
    statefulset.apps/web scaled
    [root@devops-101 ~]# kubectl get pods -w
    NAME      READY     STATUS    RESTARTS   AGE
    web-0     1/1       Running   0          5m
    web-1     1/1       Running   0          5m
    web-2     0/1       Pending   0         1s
    web-2     0/1       Pending   0         1s
因为我的集群PV不是动态供给的，所以扩容停留在等待PV的阶段，本文要按照后面的办法手工创建相应的PV。

将实例减少到2个。

    [root@devops-101 ~]# kubectl get pod
    NAME      READY     STATUS    RESTARTS   AGE
    web-0     1/1       Running   0          24m
    web-1     1/1       Running   0          23m
    web-2     1/1       Running   0          18m
    web-3     1/1       Running   0          10m
    web-4     1/1       Running   0          6m
    [root@devops-101 ~]# kubectl patch sts web -p '{"spec":{"replicas":2}}'
    statefulset.apps/web patched

# 四、更新策略

## 4.1 滚动更新

默认的更新策略，以相反的顺序依次更新Pod

4.2 Partition
4.3 金丝雀发布 Canary
金丝雀发布。

## 五、删除 StatefulSet
StatefulSet支持级连删除和非级连删除，在非级连删除模式下，仅删除StatefulSet不删除Pod，级连删除则全部删除。

非级连删除StatefulSet后，如果删除Pod，就不会重新拉起原来的Pod，而是新建一个Pod。但是如果重新创建StatefulSet，则会对现有的Pod按照规则进行重新整理。

一些分布式系统，并不希望按照顺序来管理启停Pod，因此在1.7版本之后，提供了`.spec.podManagementPolicy`这个参数，默认为OrderedReady，可以设置为`Parallel`,这样Pod的创建就不必等待，而是会同时创建、同时删除。

## 六、其他
提前创建PV及PVC。PV的创建脚本如下：

    kind: List
    apiVersion: v1
    items:
    - apiVersion: v1
      kind: PersistentVolume
      metadata:
        name: es-storage-pv-01
      spec:
        capacity:
          storage: 100Mi
        volumeMode: Filesystem
        accessModes: ["ReadWriteOnce"]
        persistentVolumeReclaimPolicy: Delete
        storageClassName: local-storage
        local:
          path: /home/es
        nodeAffinity:
          required:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - devops-102
                - devops-103
    - apiVersion: v1
      kind: PersistentVolume
      metadata:
        name: es-storage-pv-02
      spec:
        capacity:
          storage: 100Mi
        volumeMode: Filesystem
        accessModes: ["ReadWriteOnce"]
        persistentVolumeReclaimPolicy: Delete
        storageClassName: local-storage
        local:
          path: /home/es01
        nodeAffinity:
          required:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - devops-102
                - devops-103

# REF
> [Kubernetes中StatefulSet介绍](https://www.cnblogs.com/cocowool/p/kubernetes_statefulset.html)  
> [Kubernetes资源对象之Persistent Volumes](https://blog.frognew.com/2017/01/kubernetes-persistent-volumes.html)