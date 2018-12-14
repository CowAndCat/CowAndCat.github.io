---
layout: post
title: docker日常操作集合
category: docker
comments: true
---

## 一、Commit

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

    abc@BDSHYF000135355:~/Docker$ docker save -o dongle-basegpu.tar local/dongle-basegpu:1.0.0 
    // 出去上个厕所
    abc@BDSHYF000135355:~/Docker$ ls
    Dockerfile         basegpu.tar        dongle             dongle-basegpu.tar

## 二、Load 和 Save

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