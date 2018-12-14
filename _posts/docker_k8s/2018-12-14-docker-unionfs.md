---
layout: post
title: docker UnionFS
category: docker
comments: true
---

## 一、UnionFS
UnionFS是一种为Linux，FreeBSD和NetBSD操作系统设计的**把其他文件系统联合到一个联合挂载点的文件系统服务。它使用branch把不同文件系统的文件和目录“透明地”覆盖，形成一个单一一致的文件系统。**这些branches或者是read-only或者是read-write的，所以当对这个虚拟后的联合文件系统进行写操作的时候，系统是真正写到了一个新的文件中。看起来这个虚拟后的联合文件系统是可以对任何文件进行操作的，但是其实它并没有改变原来的文件，这是因为unionfs用到了一个重要的资源管理技术叫写时复制。

写时复制（copy-on-write，下文简称CoW），也叫隐式共享，是一种对可修改资源实现高效复制的资源管理技术。它的思想是，**如果一个资源是重复的，但没有任何修改，这时候并不需要立即创建一个新的资源；这个资源可以被新旧实例共享。创建新资源发生在第一次写操作，也就是对资源进行修改的时候。通过这种资源共享的方式，可以显著地减少未修改资源复制带来的消耗，但是也会在进行资源修改的时候增减小部分的开销。**

用一个经典的例子来解释一下，Knoppix，一个用于Linux演示、光盘教学和商业产品演示的Linux发行版，就是把一个CD-ROM或者DVD和一个存在在可读写设备（eg，U盘）上的叫knoppix.img的文件系统联合起来。这样任何对CD/DVD上文件的改动都会在被应用在U盘上，不改变原来的CD/DVD上的内容。

## 二、AUFS
AUFS，英文全称是Advanced multi-layered unification filesystem, 曾经也叫 Acronym multi-layered unification filesystem，Another multi-layered unification filesystem。AUFS完全重写了早期的UnionFS 1.x，其主要目的是为了可靠性和性能，并且引入了一些新的功能，比如可写分支的负载均衡。AUFS的一些实现已经被纳入UnionFS 2.x版本。

## 三、Docker是如何使用AUFS的
AUFS是Docker选用的第一种存储驱动。AUFS具有快速启动容器，高效利用存储和内存的优点，直到现在AUFS仍然是Docker支持的一种存储驱动类型。接下来介绍Docker是如何利用AUFS存储images和containers的。

### 3.1 image layer和AUFS

![image ufs](/images/201812/image_ufs.png)

**每一个Docker image都是由一系列的read-only layers组成。**image layers的内容都存储在Docker hosts filesystem的/var/lib/docker/aufs/diff目录下。而/var/lib/docker/aufs/layers目录则存储着image layer如何堆栈这些layer的metadata。

    $ docker pull ubuntu:15.04
    15.04: Pulling from library/ubuntu
    9502adfba7f1: Pull complete
    4332ffb06e4b: Pull complete
    2f937cc07b5f: Pull complete
    a3ed95caeb02: Pull complete
    Digest: sha256:2fb27e433b3ecccea2a14e794875b086711f5d49953ef173d8a03e8707f1510f
    Status: Downloaded newer image for ubuntu:15.04

    $ ls /var/lib/docker/aufs/diff
    208319b22189a2c3841bc4a4ef0df9f9238a3e832dc403133fb8ad4a6c22b01b
    9c444e426a4a0aa3ad8ff162dd7bcd4dcbb2e55bdec268b24666171904c17573
    6bb19cb345da470e015ba3f1ca049a1c27d2c57ebc205ec165d2ad8a44e148ea
    f193107618deb441376a54901bc9115f30473c1ec792b7fb3e73a98119e2cf77

    $ ls /var/lib/docker/aufs/mnt
    208319b22189a2c3841bc4a4ef0df9f9238a3e832dc403133fb8ad4a6c22b01b
    9c444e426a4a0aa3ad8ff162dd7bcd4dcbb2e55bdec268b24666171904c17573
    6bb19cb345da470e015ba3f1ca049a1c27d2c57ebc205ec165d2ad8a44e148ea
    f193107618deb441376a54901bc9115f30473c1ec792b7fb3e73a98119e2cf77

    $ cat /var/lib/docker/aufs/layers/6bb19cb345da470e015ba3f1ca049a1c27d2c57ebc205ec165d2ad8a44e148ea
    9c444e426a4a0aa3ad8ff162dd7bcd4dcbb2e55bdec268b24666171904c17573
    f193107618deb441376a54901bc9115f30473c1ec792b7fb3e73a98119e2cf77
    208319b22189a2c3841bc4a4ef0df9f9238a3e832dc403133fb8ad4a6c22b01b

接下来我们要以ubuntu:15.04镜像为基础镜像，创建一个名为changed-ubuntu的镜像。这个镜像只是在镜像的/tmp文件夹中添加一个写了“Hello world”的文件。你可以使用下面的Dockerfile来实现:

    FROM ubuntu:15.04

    RUN echo "Hello world" > /tmp/newfile

在terminal中cd到上面Dockerfile所在位置，执行docker build -t changed-ubuntu .命令来build镜像。

    $docker build -t changed-ubuntu .
    Sending build context to Docker daemon 10.75 kB
    Step 1 : FROM ubuntu:15.04
     ---> d1b55fd07600
    Step 2 : RUN echo "Hello world" > /tmp/newfile
     ---> Running in c72100f81dd1
     ---> 9d8602c9aee1
    Removing intermediate container c72100f81dd1
    Successfully built 9d8602c9aee1

然后执行docker images查看现在的镜像,可以看到新生成的changed-ubuntu。

    $docker images
    REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
    changed-ubuntu      latest              9d8602c9aee1        About a minute ago   131.3 MB
    ubuntu              15.04               d1b55fd07600        10 months ago        131.3 M

使用docker history changed-ubuntu命令可以清楚地查看到changed-ubuntu镜像使用了哪些image layers。

    $docker history changed-ubuntu
    IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
    9d8602c9aee1        4 minutes ago       /bin/sh -c echo "Hello world" > /tmp/newfile    12 B
    d1b55fd07600        10 months ago       /bin/sh -c #(nop) CMD ["/bin/bash"]             0 B
    <missing>           10 months ago       /bin/sh -c sed -i 's/^#\s*\(deb.*universe\)$/   1.879 kB
    <missing>           10 months ago       /bin/sh -c echo '#!/bin/sh' > /usr/sbin/polic   701 B
    <missing>           10 months ago       /bin/sh -c #(nop) ADD file:3f4708cf445dc1b537   131.3 MB

从输出中可以看到9d8602c9aee1 image layer位于最上层，只有12B的大小，就是由`/bin/sh -c echo "Hello world" > /tmp/newfile`命令创建的。也就是说changed-ubuntu镜像只占用了12Bytes的磁盘空间，这也证明了AUFS是如何高效使用磁盘空间的。而下面的四层image layers则是共享的构成ubuntu:15.04镜像的4个image layers。

“missing”标记的layers是自Docker 1.10之后，一个镜像的image layers的image history数据都存储在一个文件中导致，这是一个Docker官方认为的正常行为。

继续查看layers的存储信息。

    $ ls /var/lib/docker/aufs/diff
    208319b22189a2c3841bc4a4ef0df9f9238a3e832dc403133fb8ad4a6c22b01b
    9c444e426a4a0aa3ad8ff162dd7bcd4dcbb2e55bdec268b24666171904c17573
    f193107618deb441376a54901bc9115f30473c1ec792b7fb3e73a98119e2cf77
    6bb19cb345da470e015ba3f1ca049a1c27d2c57ebc205ec165d2ad8a44e148ea
    9f122dbaa103338f27bac146326af38a2bcb52f98ebb3530cac828573faa3c4e

    $ cat /var/lib/docker/aufs/diff/9f122dbaa103338f27bac146326af38a2bcb52f98ebb3530cac828573faa3c4e/tmp/newfile
    Hello world

从输出中我们可以看到/var/lib/docker/aufs/diff目录和/var/lib/docker/aufs/mnt目录均多了一个文件夹9f122dbaa。当使用cat /var/lib/docker/aufs/layers/9f122dbaa命令来查看它的metadata时可以看到，它前面的layers就是ubuntu:15.04镜像所使用的4个image layers。  
进一步的探查/var/lib/docker/aufs/diff/9f122dbaa文件夹，发现其中存储了一个/tmp/newfile文件，文件中只有一行“Hello world”。至此，我们完整地分析出了image layer和AUFS是如何通过共享文件和文件夹来实现镜像存储的。

### 3.2 container layer和AUFS

![container ufs](/images/201812/container-ufs.png)

容器的定义和镜像几乎一模一样，也是一堆层的统一视角，唯一区别在于容器的最上面那一层是可读可写的。

Docker使用AUFS的CoW技术来实现image layer共享和减少磁盘空间占用。
CoW意味着一旦某个文件只有很小的部分有改动，AUFS也需要复制整个文件。这种设计会对容器性能产生一定的影响，尤其是在待拷贝的文件很大，或者位于很多image layers的下方，或AUFS需要深度搜索目录结构树的时候。不过也不用过度担心，对于一个容器，每一个image layer最多只需要拷贝一次。后续的改动都会在第一次拷贝的container layer上进行。

**启动一个Container的时候，Docker会为其创建一个read-only的init layer，用来存储与这个容器内环境相关的内容；Docker还会为其创建一个read-write的layer用来执行所有写操作。**

Container layer的mount目录也是/var/lib/docker/aufs/mnt。Container的metadata和配置文件都存放在/var/lib/docker/containers/目录中。Container的read-write layer存储在/var/lib/docker/aufs/diff/目录下。**即使容器停止，这个可读写层仍然存在，因而重启容器不会丢失数据，只有当一个容器被删除的时候，这个可读写层才会一起删除。**

例如，启动一个changed-ubuntu的容器。

    $docker run -dit changed-ubuntu bash
    fb5939d878bb0521008d63eb06adea75e6af275855f11879dfa3992dfdaa5e3f

    $docker ps -a
    CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
    fb5939d878bb        changed-ubuntu      "bash"              28 seconds ago      Up 27 seconds                           amazing_babbage

    $ ls /var/lib/docker/aufs/diff
    208319b22189a2c3841bc4a4ef0df9f9238a3e832dc403133fb8ad4a6c22b01b
    9f122dbaa103338f27bac146326af38a2bcb52f98ebb3530cac828573faa3c4e
    f9ccf5caa9b7324f0ef112750caa14203b557d276ca08c78c23a42a949e2bfc8-init
    6bb19cb345da470e015ba3f1ca049a1c27d2c57ebc205ec165d2ad8a44e148ea
    f193107618deb441376a54901bc9115f30473c1ec792b7fb3e73a98119e2cf77
    9c444e426a4a0aa3ad8ff162dd7bcd4dcbb2e55bdec268b24666171904c17573
    f9ccf5caa9b7324f0ef112750caa14203b557d276ca08c78c23a42a949e2bfc8

查看/var/lib/docker/aufs/diff目录发现，下面多了两个文件夹，f9ccf5caa9就是Docker为容器创建的read-write layer，而f9ccf5caa9..fc8-init则是Docker为容器创建的read-only的init layer。

最后提下AUFS如何为Container删除一个文件。如果要删除file1，AUFS会在container的read-write层生成一个.wh.file1的文件来隐藏所有read-only层的file1文件。  
至此，我们已清楚地描述和验证了Docker是如何使用AUFS来管理container layers的。

## 四、自己动手写AUFS

略

## REF 
> [Docker之Linux UnionFS](http://www.cnblogs.com/ilinuxer/p/6188654.html)