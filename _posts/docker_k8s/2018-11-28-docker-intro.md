---
layout: post
title: Docker 是什么
category: docker
comments: false
---
# 一、Docker是什么

Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。容器是完全使用沙箱机制，相互之间不会有任何接口。

作为一种新兴的虚拟化方式，Docker 跟传统的虚拟化方式相比具有众多的优势。传统虚拟机技术是虚拟出一套硬件后，在其上运行一个完整操作系统，在该系统上再运行所需应用进程；而容器内的应用进程直接运行于宿主的内核，容器内没有自己的内核，而且也没有进行硬件虚拟。因此容器要比传统虚拟机更为轻便。

Docker通常用于如下场景：

- web应用的自动化打包和发布；
- 自动化测试和持续集成、发布；
- 在服务型环境中部署和调整数据库或其他的后台应用；
- 从头编译或者扩展现有的OpenShift或Cloud Foundry平台来搭建自己的PaaS环境。

Docker 包括三个基本概念：镜像、容器、仓库。

## 1.1 Docker 镜像
Docker 镜像是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像不包含任何动态数据，其内容在构建之后也不会被改变。

对于 Linux 而言，内核启动后，会挂载 root 文件系统为其提供用户空间支持。而 Docker 镜像（Image），就相当于是一个 root 文件系统。

【分层存储】因为镜像包含操作系统完整的 root 文件系统，其体积往往是庞大的，因此在 Docker 设计时，就充分利用 Union FS 的技术，将其设计为分层存储的架构。所以严格来说，镜像并非是像一个 ISO 那样的打包文件，镜像只是一个虚拟的概念，其实际体现并非由一个文件组成，而是由一组文件系统组成，或者说，由多层文件系统联合组成。

镜像构建时，会一层层构建，前一层是后一层的基础。每一层构建完就不会再发生改变，后一层上的任何改变只发生在自己这一层。

## 1.2 Docker 容器
镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的 类 和 实例 一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。

容器的实质是进程，但与直接在宿主执行的进程不同，容器进程运行于属于自己的独立的 命名空间。

容器内的进程是运行在一个隔离的环境里，使用起来，就好像是在一个独立于宿主的系统下操作一样。这种特性使得容器封装的应用比直接在宿主运行更加安全。也因为这种隔离的特性，很多人初学 Docker 时常常会混淆容器和虚拟机。

按照 Docker 最佳实践的要求，容器不应该向其存储层内写入任何数据，容器存储层要保持无状态化。所有的文件写入操作，都应该使用 数据卷（Volume）、或者绑定宿主目录，在这些位置的读写会跳过容器存储层，直接对宿主（或网络存储）发生读写，其性能和稳定性更高。

数据卷的生存周期独立于容器，容器消亡，数据卷不会消亡。因此，使用数据卷后，容器删除或者重新运行之后，数据却不会丢失。

## 1.3 Docker 仓库

镜像构建完成后，可以很容易的在当前宿主机上运行，但是，如果需要在其它服务器上使用这个镜像，我们就需要一个集中的存储、分发镜像的服务，Docker Registry 就是这样的服务。

一个 Docker Registry 中可以包含多个仓库（Repository）；每个仓库可以包含多个标签（Tag）；每个标签对应一个镜像。

通常，一个仓库会包含同一个软件不同版本的镜像，而标签就常用于对应该软件的各个版本。我们可以通过 <仓库名>:<标签> 的格式来指定具体是这个软件哪个版本的镜像。如果不给出标签，将以 latest 作为默认标签。

以 Ubuntu 镜像 为例，ubuntu 是仓库的名字，其内包含有不同的版本标签，如，14.04, 16.04。我们可以通过 ubuntu:14.04，或者 ubuntu:16.04 来具体指定所需哪个版本的镜像。如果忽略了标签，比如 ubuntu，那将视为 ubuntu:latest。

仓库名经常以 两段式路径 形式出现，比如 jwilder/nginx-proxy，前者往往意味着 Docker Registry 多用户环境下的用户名，后者则往往是对应的软件名。但这并非绝对，取决于所使用的具体 Docker Registry 的软件或服务。

Docker Registry可以分公开和私有的，类似于git上的项目。  

国内也有一些云服务商提供类似于 Docker Hub 的公开服务。比如 时速云镜像仓库、网易云镜像服务、DaoCloud 镜像市场、阿里云镜像库 等。

Docker 官方提供了 Docker Registry 镜像，可以直接使用做为私有 Registry 服务。

# 二、Docker 探索

Docker系统有两个程序：docker服务端和docker客户端。其中docker服务端是一个服务进程，管理着所有的容器。docker客户端则扮演着docker服务端的远程控制器，可以用来控制docker的服务端进程。大部分情况下，docker服务端和客户端运行在一台机器上。

### 2.1 Docker 搜索

Docker官方网站专门有一个页面来存储所有可用的镜像，网址是：index.docker.io。可以通过浏览这个网页来查找你想要使用的镜像，或者使用命令行的工具来检索。

命令行的格式为：docker search 镜像名字

	$docker search tutorial

### 2.2 Docker 下载
下载镜像的命令非常简单，使用docker pull命令即可。

	$docker pull learn/tutorial

要想列出已经下载下来的镜像，可以使用 docker image ls 命令。

	 $docker image ls

这里显示的是镜像下载到本地后，展开的大小。Docker Hub 中显示的体积是压缩后的体积。

另外一个需要注意的问题是，docker image ls 列表中的镜像体积总和并非是所有镜像实际硬盘消耗。由于 Docker 镜像是多层存储结构，并且可以继承、复用，因此不同镜像可能会因为使用相同的基础镜像，从而拥有共同的层。由于 Docker 使用 Union FS，相同的层只需要保存一份即可，因此实际镜像硬盘占用空间很可能要比这个列表镜像大小的总和要小的多。

通过以下命令来便捷的查看镜像、容器、数据卷所占用的空间。

	$ docker system df

### 2.3 Docker 运行

docker容器可以理解为在沙盒中运行的进程。这个沙盒包含了该进程运行所必须的资源，包括文件系统、系统类库、shell 环境等等。但这个沙盒默认是不会运行任何程序的。你需要在沙盒中运行一个进程来启动某一个容器。这个进程是该容器的唯一进程，所以当该进程结束的时候，容器也会完全的停止。

docker run命令有两个参数，一个是镜像名，一个是要在镜像中运行的命令
	
	$docker run learn/tutorial echo "hello word"

下面的命令则启动一个 bash 终端，允许用户进行交互。

	$ docker run -t -i ubuntu:14.04 /bin/bash
	root@af8bae53bdd3:/#
其中，-t 选项让Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上， -i 则让容器的标准输入保持打开。（可以合成-it）

当利用 docker run 来创建容器时，Docker 在后台运行的标准操作包括：

- 检查本地是否存在指定的镜像，不存在就从公有仓库下载
- 利用镜像创建并启动一个容器
- 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
- 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
- 从地址池配置一个 ip 地址给容器
- 执行用户指定的应用程序
- 执行完毕后容器被终止

另外，可以利用 docker container start 命令，直接将一个已经终止的容器启动运行。

### 2.4 在容器中安装新的程序

下一步我们要做的事情是在容器里面安装一个简单的程序(ping)。我们之前下载的tutorial镜像是基于ubuntu的，所以你可以使用ubuntu的apt-get命令来安装ping程序： 

	apt-get install -y ping。

备注：apt-get 命令执行完毕之后，容器就会停止，但对容器的改动不会丢失。

目标：在learn/tutorial镜像里面安装ping程序。
提示：在执行apt-get 命令的时候，要带上-y参数。如果不指定-y参数的话，apt-get命令会进入交互模式，需要用户输入命令来进行确认，但在docker环境中是无法响应这种交互的。

正确的命令：

	$docker run learn/tutorial apt-get install -y ping

### 2.5 保存对容器的修改

当你对某一个容器做了修改之后（通过在容器中运行某一个命令），可以把对容器的修改保存下来，这样下次可以从保存后的最新状态运行该容器。docker中保存状态的过程称之为committing，它保存的新旧状态之间的区别，从而产生一个新的版本。

目标：首先使用 docker ps -l命令获得安装完ping命令之后容器的id。然后把这个镜像保存为learn/ping。

提示：

1. 运行docker commit，可以查看该命令的参数列表。
2. 你需要指定要提交保存容器的ID。(译者按：通过docker ps -l 命令获得)
3. 无需拷贝完整的id，通常来讲最开始的三至四个字母即可区分。（译者按：非常类似git里面的版本号)

正确的命令：

	$ docker commit 698 learn/ping

不要使用 docker commit 定制镜像，定制镜像应该使用 Dockerfile 来完成。

### 2.6 运行新的镜像

在新的镜像中运行`ping www.google.com`命令。

提示：一定要使用新的镜像名learn/ping来运行ping命令。(译者按：最开始下载的learn/tutorial镜像中是没有ping命令的)

正确的命令：

	$docker run lean/ping ping www.google.com

### 2.7 检查运行中的镜像

- 使用 docker ps命令可以查看所有正在运行中的容器列表
- 使用 docker inspect命令我们可以查看更详细的关于某一个容器的信息。

目标：查找某一个运行中容器的id，然后使用docker inspect命令查看容器的信息。

提示：可以使用镜像id的前面部分，不需要完整的id。

正确的命令：

	$ docker inspect efe

### 2.8 发布docker镜像

现在已经验证了新镜像可以正常工作，下一步可以将其发布到官方的索引网站。还记得我们最开始下载的learn/tutorial镜像吧，我们也可以把我们自己编译的镜像发布到索引页面，一方面可以自己重用，另一方面也可以分享给其他人使用。

目标：把learn/ping镜像发布到docker的index网站。

提示：

1. docker images命令可以列出所有安装过的镜像。
2. docker push命令可以将某一个镜像发布到官方网站。
3. 你只能将镜像发布到自己的空间下面。这个模拟器登录的是learn帐号。

预期的命令：

	$ docker push learn/ping

### 2.9 Dockerfile 请参见笔者的另一篇文章。

### 2.10 后台运行

注： 容器是否会长久运行，是和 docker run 指定的命令有关，和 -d 参数无关。

使用 -d 参数运行容器，只是把输出给重定向了。

	$ docker run -d ubuntu:17.10 /bin/sh -c "while true; do echo hello world; sleep 1; done"
	77b2dc01fe0f3f1265df143181e7b9af5e05279a884f4776ee75350ea9d8017a

此时容器会在后台运行并不会把输出的结果 (STDOUT) 打印到宿主机上面(输出结果可以用 docker logs 查看)。

使用 -d 参数启动后会返回一个唯一的 id，也可以通过 docker container ls 命令来查看容器信息。

	$ docker container ls
	CONTAINER ID  IMAGE         COMMAND               CREATED        STATUS       PORTS NAMES
	77b2dc01fe0f  ubuntu:17.10  /bin/sh -c 'while tr  2 minutes ago  Up 1 minute        agitated_wright

要获取容器的输出信息，可以通过 docker container logs 命令。

	$ docker container logs [container ID or NAMES]
	hello world
	hello world
	hello world

### 2.11 终止容器

可以使用 docker container stop 来终止一个运行中的容器。

此外，当 Docker 容器中指定的应用终结时，容器也自动终止。

例如对于上一章节中只启动了一个终端的容器，用户通过 exit 命令或 Ctrl+d 来退出终端时，所创建的容器立刻终止。

终止状态的容器可以用 docker container ls -a 命令看到。例如

	docker container ls -a
	CONTAINER ID        IMAGE                    COMMAND                CREATED             STATUS                          PORTS               NAMES
	ba267838cc1b        ubuntu:14.04             "/bin/bash"            30 minutes ago      Exited (0) About a minute ago                       trusting_newton
	98e5efa7d997        training/webapp:latest   "python app.py"        About an hour ago   Exited (0) 34 minutes ago                           backstabbing_pike

处于终止状态的容器，可以通过 docker container start 命令来重新启动。

此外，docker container restart 命令会将一个运行态的容器终止，然后再重新启动它。

### 2.12 进入容器
某些时候需要进入容器进行操作，包括使用 docker attach 命令或 docker exec 命令，推荐大家使用 docker exec 命令，原因会在下面说明。

docker attach：

	$ docker run -dit ubuntu
	243c32535da7d142fb0e6df616a3c3ada0b8ab417937c853a9e1c251f499f550

	$ docker container ls
	CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
	243c32535da7        ubuntu:latest       "/bin/bash"         18 seconds ago      Up 17 seconds                           nostalgic_hypatia

	$ docker attach 243c
	root@243c32535da7:/#

如果从这个 stdin 中 exit，会导致容器的停止。

docker exec：

后边可以跟多个参数，这里主要说明 -i -t 参数。

只用 -i 参数时，由于没有分配伪终端，界面没有我们熟悉的 Linux 命令提示符，但命令执行结果仍然可以返回。

当 -i -t 参数一起使用时，则可以看到我们熟悉的 Linux 命令提示符。

	$ docker run -dit ubuntu
	69d137adef7a8a689cbcb059e94da5489d3cddd240ff675c640c8d96e84fe1f6

	$ docker container ls
	CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
	69d137adef7a        ubuntu:latest       "/bin/bash"         18 seconds ago      Up 17 seconds                           zealous_swirles

	$ docker exec -i 69d1 bash
	ls
	bin
	boot
	dev
	...

	$ docker exec -it 69d1 bash
	root@69d137adef7a:/#

如果从这个 stdin 中 exit，不会导致容器的停止。这就是为什么推荐大家使用 docker exec 的原因。

### 2.13 导出和导入容器

如果要导出本地某个容器，可以使用 docker export 命令。

	$ docker container ls -a
	CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                    PORTS               NAMES
	7691a814370e        ubuntu:14.04        "/bin/bash"         36 hours ago        Exited (0) 21 hours ago                       test
	$ docker export 7691a814370e > ubuntu.tar

使用 docker import 从容器快照文件中再导入为镜像，例如

	$ cat ubuntu.tar | docker import - test/ubuntu:v1.0
	$ docker image ls
	REPOSITORY          TAG                 IMAGE ID            CREATED              VIRTUAL SIZE
	test/ubuntu         v1.0                9d37a6082e97        About a minute ago   171.3 MB

此外，也可以通过指定 URL 或者某个目录来导入，例如：

	$ docker import http://example.com/exampleimage.tgz example/imagerepo

### 2.14 删除容器

可以使用 docker container rm 来删除一个处于终止状态的容器。例如

	$ docker container rm  trusting_newton
	trusting_newton
如果要删除一个运行中的容器，可以添加 -f 参数。Docker 会发送 SIGKILL 信号给容器。

清理所有处于终止状态的容器

用 docker container ls -a 命令可以查看所有已经创建的包括终止状态的容器，如果数量太多要一个个删除可能会很麻烦，用下面的命令可以清理掉所有处于终止状态的容器。

	$ docker container prune

# 三、注意事项

### 3.1 慎用 docker commit

使用 docker commit 命令虽然可以比较直观的帮助理解镜像分层存储的概念，但是实际环境中并不会这样使用。

首先，如果仔细观察之前的 docker diff webserver 的结果，你会发现除了真正想要修改的 /usr/share/nginx/html/index.html 文件外，由于命令的执行，还有很多文件被改动或添加了。这还仅仅是最简单的操作，如果是安装软件包、编译构建，那会有大量的无关内容被添加进来，如果不小心清理，将会导致镜像极为臃肿。

此外，使用 docker commit 意味着所有对镜像的操作都是黑箱操作，生成的镜像也被称为**黑箱镜像**，换句话说，就是除了制作镜像的人知道执行过什么命令、怎么生成的镜像，别人根本无从得知。而且，即使是这个制作镜像的人，过一段时间后也无法记清具体在操作的。虽然 docker diff 或许可以告诉得到一些线索，但是远远不到可以确保生成一致镜像的地步。这种黑箱镜像的维护工作是非常痛苦的。

而且，回顾之前提及的镜像所使用的分层存储的概念，除当前层外，之前的每一层都是不会发生改变的，换句话说，任何修改的结果仅仅是在当前层进行标记、添加、修改，而不会改动上一层。如果使用 docker commit 制作镜像，以及后期修改的话，每一次修改都会让镜像更加臃肿一次，所删除的上一层的东西并不会丢失，会一直如影随形的跟着这个镜像，即使根本无法访问到。这会让镜像更加臃肿。




# REF
> [docker入门教程](http://www.docker.org.cn/book/docker/docker-getting-started-14.html)
> [Docker Hello World](http://www.runoob.com/docker/docker-hello-world.html)