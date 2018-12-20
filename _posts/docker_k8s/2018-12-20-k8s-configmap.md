---
layout: post
title: Kubernetes文件挂载
category: docker
comments: true
---

## 一、应用场景

应用场景：

- 替换镜像中配置文件
- 替换镜像中的环境变量

当ConfigMap以数据卷的形式挂载进Pod的时，这时更新ConfigMap（或删掉重建ConfigMap），Pod内挂载的配置信息会热更新。  
这时可以增加一些监测配置文件变更的脚本，然后reload对应服务。

关于热更新，需要注意：

- 使用该 ConfigMap 挂载的 Env 不会同步更新
- 使用该 ConfigMap 挂载的 Volume 中的数据需要一段时间（实测大概10秒）才能同步更新

因为ENV 是在容器启动的时候注入的，启动之后 kubernetes 就不会再改变环境变量的值，且同一个 namespace 中的 pod 的环境变量是不断累加的。为了更新容器中使用 ConfigMap 挂载的配置，可以通过滚动更新 pod 的方式来强制重新挂载 ConfigMap，也可以在更新了 ConfigMap 后，先将副本数设置为 0，然后再扩容。


## 二、操作

### 2.1 替换配置文件

首先，将文件制作成一个configmap：

    kubectl create configmap image-config --from-file=License_enc

image-config是configmap的配置，会在后面的yaml应用到； License_enc可以是一个二进制文件。

然后，修改原先部署的yaml文件：

    apiVersion: v1
    kind: Service
    metadata:
      name: some-service
      labels:
        app: some-app
    spec:
      selector:
        app: some-app
      ports:
        - name: some-app
          port: 5005
          nodePort: 35001
      type: NodePort
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: some-app
    spec:
      selector:
        matchLabels:
          app: some-app
      replicas: 1
      template:
        metadata:
          labels:
            app: some-app
        spec:
          containers:
          - name: some-app
            image: namespace/some-app:1.0
            env:
            - name: PROCESS_SERVER_PORT
              value: "5005"
            - name: SOME_SERVER
              value: "192.0.0.1"
            volumeMounts:
            - name: license
              subPath: License_enc
              mountPath: /root/app/License_enc
          volumes:
          - name: license
            configMap:
              name: image-config

主要增加了这几行：
    
        volumeMounts:
        - name: license
          subPath: License_enc
          mountPath: /root/app/License_enc
      volumes:
      - name: license
        configMap:
          name: image-config

license用于在yaml配置文件里定位volumes配置，image-config就是上面制作的保存在k8s里的configmap，configmap里的文件会替换/root/app/License_enc 文件。  
subPath必须要与configmap中的key同名。

### 2.2 使用成环境变量

可以使用configmap中指定的key或使用configmap中所有的key。

(1)使用valueFrom、configMapKeyRef、name、key指定要用的key:

    apiVersion: v1
    kind: Pod
    metadata:
      name: dapi-test-pod
    spec:
      containers:
        - name: test-container
          image: k8s.gcr.io/busybox
          command: [ "/bin/sh", "-c", "env" ]
          env:
            - name: SPECIAL_LEVEL_KEY
              valueFrom:
                configMapKeyRef:
                  name: special-config
                  key: special.how
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: env-config
                  key: log_level
      restartPolicy: Never

(2)还可以通过envFrom、configMapRef、name使得configmap中的所有key/value对都自动变成环境变量：

    apiVersion: v1
    kind: Pod
    metadata:
      name: dapi-test-pod
    spec:
      containers:
        - name: test-container
          image: k8s.gcr.io/busybox
          command: [ "/bin/sh", "-c", "env" ]
          envFrom:
          - configMapRef:
              name: special-config
      restartPolicy: Never

(3)通过挂载Volumes设置环境变量

    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: nginx-configmap
    spec:
      replicas: 1
      template:
        metadata:
          labels:
            app: nginx-configmap
        spec:
          containers:
          - name: nginx-configmap
            image: nginx
            ports:
            - containerPort: 80
            volumeMounts:     
            - name: config-volume4
              mountPath: /tmp/config4
          volumes:
          - name: config-volume4
            configMap:
              name: test-config4

假如不想以key名作为配置文件名可以引入items 字段，在其中逐个指定要用相对路径path替换的key：

      volumes:
      - name: config-volume4
        configMap:
          name: test-config4
          items:
          - key: my.cnf
            path: mysql-key
          - key: cache_host
            path: cache-host

## 三、相关命令

### 3.1 创建ConfigMap
方式有4种：

- 通过直接在命令行中指定configmap参数创建，即`--from-literal`

        kubectl create configmap test-config1 --from-literal=db.host=10.5.10.116 --from-listeral=db.port='3306'

- 通过指定文件创建，即将一个配置文件创建为一个ConfigMap`--from-file=<文件>`

        kubectl create configmap test-config2 --from-file=./app.properties

    假如不想configmap中的key为默认的文件名，还可以在创建时指定key名字：

        kubectl create configmap game-config-3 --from-file=<my-key-name>=<path-to-file>

- 通过指定目录创建，即将一个目录下的所有配置文件创建为一个ConfigMap，`--from-file=<目录>`

        kubectl create configmap test-config4 --from-file=./configs

    指定目录创建时configmap内容中的各个文件会创建一个key/value对，key是文件名，value是文件内容。  
    注意： 指定目录时只会识别其中的文件，忽略子目录。

- 事先写好标准的configmap的yaml文件，然后`kubectl create -f` 创建,一个ConfigMap类型的文件类似这样：

        apiVersion: v1
        kind: ConfigMap
        metadata:
            name: test-config5
            namespace: default
        data:
            cache_host: memcached-gcxt
            cache_port: "33303"
            cache_prefix: gcxt
            my.cnf: |
                [mysqld]
                some_key=some_value


### 3.2 查看ConfigMap

输出yaml格式：
    
    kubectl get configmap test-config1 -o yaml

如果是二进制，会转成可见的字符串（怀疑是base64）

还支持的格式有：custom-columns,custom-columns-file,go-template,go-template-file,json,jsonpath,jsonpath-file,name,template,templatefile,wide

### 3.3 修改ConfigMap

    kubectl edit configmap ...

过一会pod内部的配置也会刷新。但是如果在容器内部修改挂进去的配置文件后，过一会内容会再次被刷新为原始configmap内容。

### 3.4 删除ConfigMap

    kubectl delete configmap CONFIG_NAME

删除configmap后原pod不受影响；然后再删除pod后，重启的pod的events会报找不到cofigmap的volume.


## REF
> [Kubernetes 1.2 新功能介绍：ConfigMap](https://blog.csdn.net/u010884123/article/details/57416537)

