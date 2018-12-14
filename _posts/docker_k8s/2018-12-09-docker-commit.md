---
layout: post
title: docker日常操作集合
category: docker
comments: true
---

## 一、commit

基于已有的docker容器，做一新的dokcer image.

    $ docker commit <container_id> <image_name>

当你对某一个容器做了修改之后（通过在容器中运行某一个命令），可以把对容器的修改保存下来，这样下次可以从保存后的最新状态运行该容器。docker中保存状态的过程称之为committing，它保存的新旧状态之间的区别，从而产生一个新的版本。

目标： 首先使用docker ps -l命令获得安装完ping命令之后容器的id。然后把这个镜像保存为learn/ping。

提示：运行docker commit，可以查看该命令的参数列表。
你需要指定要提交保存容器的ID。(译者按：通过docker ps -l 命令获得)
无需拷贝完整的id，通常来讲最开始的三至四个字母即可区分。（译者按：非常类似git里面的版本号)

正确的命令：

    $docker ps -l
    $docker commit 698 learn/ping

实操

    abc@BDSHYF000135355:~/Docker$ docker ps
    CONTAINER ID        IMAGE                                            COMMAND             CREATED             STATUS              PORTS               NAMES
    377993ef5f69        registry.baidu-int.com/acu-voice/basegpu:1.0.0   "bash"              9 minutes ago       Up 9 minutes                            musing_keller
    
    abc@BDSHYF000135355:docker run -it registry.baidu-int.com/acu-voice/basegpu:1.0.0 bash
    // 一顿非常华丽的操作
    // 退出
    [root@377993ef5f69 packet]# exit
    exit
    
    abc@BDSHYF000135355:~/Docker$ docker ps -l
    CONTAINER ID        IMAGE                                            COMMAND             CREATED             STATUS                      PORTS               NAMES
    377993ef5f69        registry.baidu-int.com/acu-voice/basegpu:1.0.0   "bash"              11 minutes ago      Exited (0) 12 seconds ago                       musing_keller
    
    abc@BDSHYF000135355:~/Docker$ docker commit 377993ef5f69 local/dongle-basegpu:1.0.0
    sha256:fe9c80b2c7559b5698765cc8d7255608c6fa8c24863ef3b6a1fac4c04c4aa54d
    
    abc@BDSHYF000135355:~/Docker$ docker images
    REPOSITORY                                          TAG                 IMAGE ID            CREATED             SIZE
    local/dongle-basegpu                                1.0.0               fe9c80b2c755        41 seconds ago      1.06GB
    <none>                                              <none>              7854f189b179        12 hours ago        916MB
    registry.baidu.com/public_team/safedongle-centos7   0.8                 be417a3daa9a        2 weeks ago         365MB
    registry.baidu-int.com/acu-voice/basegpu            1.0.0               363478e3efb1        6 months ago        916MB

在列出的所有顶层（top-level）镜像信息中，可以看到几个字段信息

- REPOSITORY: 来自于哪个仓库，比如 ubuntu
- TAG: 镜像的标记，比如 14.04
- IMAGE ID: 它的 ID 号（唯一）
- CREATED: 创建时间
- SIZE: 镜像大小

    abc@BDSHYF000135355:~/Docker$ docker save -o dongle-basegpu.tar local/dongle-basegpu:1.0.0 
    // 出去上个厕所
    abc@BDSHYF000135355:~/Docker$ ls
    Dockerfile         basegpu.tar        dongle             dongle-basegpu.tar

## 二、load、save和export

查看镜像：

    docker images

保存镜像

    docker save spring-boot-docker -o spring-boot-docker.tar

加载镜像

    docker load -i spring-boot-docker.tar
    // 或
    docker load < spring-boot-docker.tar

跑起来

    docker run -p 8081:8080 -d spring-boot-docker

容器导出
    
    docker export <container-id>

创建一个tar文件，并且移除了元数据和不必要的层，将多个层整合成了一个层，只保存了当前统一视角看到
的内容。
**expoxt后的容器再import到Docker中，只有一个容器当前状态的镜像；而save后的镜像则不同，
它能够看到这个镜像的历史镜像。**

## 三、docker history 
这个命令可以查看镜像使用了哪些image layers

    docker history IMAGE

## 四、pull 和 push
获取镜像

    docker pull centos:centos6

实际上相当于 docker pull registry.hub.docker.com/centos:centos6
命令，即从注册服务器 registry.hub.docker.com 中的 centos 仓库来下载标记为 centos6 的镜像.

上传镜像
    
    ocker push hainiu/httpd:1.0

用户可以通过 docker push 命令，把自己创建的镜像上传到仓库中来共享。上例中，用户将镜像推送自己的镜像到仓库中。

## 五、容器相关

### 5.1 启动容器

    docker start <container-id>

Docker start命令为容器文件系统创建了一个进程隔离空间。注意，每一个容器只能够有一个进程隔离空间。

运行实例：

    #通过名字启动
    $ docker start -i centos6_container

    ＃通过容器ID启动
    $ docker start -i b3cd0b47fe3d

### 5.2 进入容器

    docker exec <container-id>

在当前容器中执行新命令，如果增加 -it参数运行bash 就和登录到容器效果一样的。

    docker exec -it centos6_container bash

### 5.3 其他  
停止容器

    docker stop <container-id>

删除容器#
    
    docker rm <container-id>

(删除镜像: `docker rmi <image-id>`)

运行容器
    
    docker run <image-id>

docker run就是docker create和docker start两个命令的组合,支持参数也是一致的。
如果指定容器名字时，容器已经存在会报错,可以增加 `--rm` 参数实现容器退出时自动删除。

运行示例:

    docker create -it --rm --name centos6_container centos:centos6

## 六、inspect
    
    docker inspect <container-id> or <image-id>

docker inspect命令会提取出容器或者镜像最顶层的元数据。