---
layout: post
title: Dockerfile定制镜像
category: docker
comments: false
---

# 一、使用 Dockerfile 定制镜像

镜像的定制实际上就是定制每一层所添加的配置、文件。

如果我们可以把每一层修改、安装、构建、操作的命令都写入一个脚本，用这个脚本来构建、定制镜像，那么之前提及的无法重复的问题、镜像构建透明性的问题、体积的问题就都会解决。这个脚本就是 Dockerfile。

Dockerfile 是一个文本文件，其内包含了一条条的指令(Instruction)，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

# 二、语法

一个Dockerfile栗子：

	FROM nginx
	RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html

这个 Dockerfile 很简单，一共就两行。涉及到了两条指令，FROM 和 RUN。

## 2.1 FROM 指定基础镜像

FROM 就是指定基础镜像，因此一个 Dockerfile 中 FROM 是必备的指令，并且必须是第一条指令。

在 Docker Store 上有非常多的高质量的官方镜像，有可以直接拿来使用的服务类的镜像，如 nginx、redis、mongo、mysql、httpd、php、tomcat 等；也有一些方便开发、构建、运行各种语言应用的镜像，如 node、openjdk、python、ruby、golang 等。可以在其中寻找一个最符合我们最终目标的镜像为基础镜像进行定制。

除了选择现有镜像为基础镜像外，Docker 还存在一个特殊的镜像，名为 scratch。这个镜像是虚拟的概念，并不实际存在，它表示一个空白的镜像。
	
	FROM scratch
	...

## 2.2 RUN 执行命令

RUN 指令是用来执行命令行命令的。由于命令行的强大能力，RUN 指令在定制镜像时是最常用的指令之一。其格式有两种：

- shell 格式：`RUN <命令>`，就像直接在命令行中输入的命令一样。刚才写的 Dockerfile 中的 RUN 指令就是这种格式。

		RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html

- exec 格式：RUN ["可执行文件", "参数1", "参数2"]，这更像是函数调用中的格式。

之前说过，Dockerfile 中每一个指令都会建立一层，RUN 也不例外。  

每一个 RUN 的行为，就和刚才我们手工建立镜像的过程一样：新建立一层，在其上执行这些命令，执行结束后，commit 这一层的修改，构成新的镜像。

Union FS 是有最大层数限制的，比如 AUFS，曾经是最大不得超过 42 层，现在是不得超过 127 层。

最好是只有一个RUN（代码目的是编译、安装 redis 可执行文件）：

	FROM debian:jessie

	RUN buildDeps='gcc libc6-dev make' \
	    && apt-get update \
	    && apt-get install -y $buildDeps \
	    && wget -O redis.tar.gz "http://download.redis.io/releases/redis-3.2.5.tar.gz" \
	    && mkdir -p /usr/src/redis \
	    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
	    && make -C /usr/src/redis \
	    && make -C /usr/src/redis install \
	    && rm -rf /var/lib/apt/lists/* \
	    && rm redis.tar.gz \
	    && rm -r /usr/src/redis \
	    && apt-get purge -y --auto-remove $buildDeps

Dockerfile 支持 Shell 类的行尾添加 \ 的命令换行方式，以及行首 # 进行注释的格式。

此外，还可以看到这一组命令的最后添加了清理工作的命令，删除了为了编译构建所需要的软件，清理了所有下载、展开的文件，并且还清理了 apt 缓存文件。这是很重要的一步，镜像是多层存储，每一层的东西并不会在下一层被删除，会一直跟随着镜像。因此镜像构建时，一定要确保每一层只添加真正需要添加的东西，任何无关的东西都应该清理掉。

很多人初学 Docker 制作出了很臃肿的镜像的原因之一，就是忘记了每一层构建的最后一定要清理掉无关文件。

在撰写 Dockerfile 的时候，要经常提醒自己，这并不是在写 Shell 脚本，而是在定义每一层该如何构建。

## 2.3 构建镜像

在 Dockerfile 文件所在目录执行：

	$ docker build -t nginx:v3 .

这里我们使用了 docker build 命令进行镜像构建。其格式为：

	docker build [选项] <上下文路径/URL/->

 -t 标记来添加 tag，指定新的镜像的用户信息。在这里我们指定了最终镜像的名称 -t nginx:v3。
 构建成功后，我们可以像之前运行 nginx:v2 那样来运行这个镜像，其结果会和 nginx:v2 一样。

这里，`.`是镜像构建的上下文（Context）路径。（并不是单纯地指Dockerfile所在的路径）这要从build的工作原理说起。

docker build 的工作原理：  
Docker 在运行时分为 Docker 引擎（也就是服务端守护进程）和客户端工具。Docker 的引擎提供了一组 REST API，被称为 Docker Remote API，而如 docker 命令这样的客户端工具，则是通过这组 API 与 Docker 引擎交互，从而完成各种功能。因此，虽然表面上我们好像是在本机执行各种 docker 功能，但实际上，一切都是使用的远程调用形式在服务端（Docker 引擎）完成。也因为这种 C/S 设计，让我们操作远程服务器的 Docker 引擎变得轻而易举。

docker build 命令构建镜像，其实并非在本地构建，而是在服务端，也就是 Docker 引擎中构建的。那么在这种客户端/服务端的架构中，如何才能让服务端获得本地文件呢？

这就引入了上下文的概念。当构建的时候，用户会指定构建镜像上下文的路径，docker build 命令得知这个路径后，会将路径下的所有内容打包，然后上传给 Docker 引擎。这样 Docker 引擎收到这个上下文包后，展开就会获得构建镜像所需的一切文件。

如果在 Dockerfile 中这么写：

	COPY ./package.json /app/

这并不是要复制执行 docker build 命令所在的目录下的 package.json，也不是复制 Dockerfile 所在目录下的 package.json，而是复制 上下文（context） 目录下的 package.json。


一般来说，应该会将 Dockerfile 置于一个空目录下，或者项目根目录下。如果该目录下没有所需文件，那么应该把所需文件复制一份过来。如果目录下有些东西确实不希望构建时传给 Docker 引擎，那么可以用 .gitignore 一样的语法写一个 .dockerignore，该文件是用于剔除不需要作为上下文传递给 Docker 引擎的。

在默认情况下，如果不额外指定 Dockerfile 的话，会将上下文目录下的名为 Dockerfile 的文件作为 Dockerfile。实际上 Dockerfile 的文件名并不要求必须为 Dockerfile,可以用 -f ../Dockerfile.php 参数指定上层路径的某个文件作为 Dockerfile。

## 2.4 其他docker build用法

- 直接用 Git repo 进行构建。
docker build 还支持从 URL 构建，比如可以直接从 Git repo 中构建

		$ docker build https://github.com/twang2218/gitlab-ce-zh.git#:8.14
		docker build https://github.com/twang2218/gitlab-ce-zh.git\#:8.14
		Sending build context to Docker daemon 2.048 kB
		Step 1 : FROM gitlab/gitlab-ce:8.14.0-ce.0
		8.14.0-ce.0: Pulling from gitlab/gitlab-ce
		aed15891ba52: Already exists
		773ae8583d14: Already exists
		...

- 用给定的 tar 压缩包构建
	
		$ docker build http://server/context.tar.gz

- 从标准输入中读取 Dockerfile 进行构建

		docker build - < Dockerfile
	或

		cat Dockerfile | docker build -

	如果标准输入传入的是文本文件，则将其视为 Dockerfile，并开始构建。这种形式由于直接从标准输入中读取 Dockerfile 的内容，它没有上下文，因此不可以像其他方法那样可以将本地文件 COPY 进镜像之类的事情。

- 从标准输入中读取上下文压缩包进行构建

		$ docker build - < context.tar.gz

	如果发现标准输入的文件格式是 gzip、bzip2 以及 xz 的话，将会使其为上下文压缩包，直接将其展开，将里面视为上下文，并开始构建。		

## 2.5 COPY 和 ADD语法

COPY 指令将从构建上下文目录中 <源路径> 的文件/目录复制到新的一层的镜像内的 <目标路径> 位置。比如：

	COPY package.json /usr/src/app/
在使用该指令的时候还可以加上 --chown=<user>:<group> 选项来改变文件的所属用户及所属组。

	COPY --chown=55:mygroup files* /mydir/
	COPY --chown=bin files* /mydir/

ADD 指令和 COPY 的格式和性质基本一致。但是在 COPY 基础上增加了一些功能。

如果 <源路径> 为一个 tar 压缩文件的话，压缩格式为 gzip, bzip2 以及 xz 的情况下，ADD 指令将会自动解压缩这个压缩文件到 <目标路径> 去。

	FROM scratch
	ADD ubuntu-xenial-core-cloudimg-amd64-root.tar.gz /
	...

在 Docker 官方的 Dockerfile 最佳实践文档 中要求，尽可能的使用 COPY，因为 COPY 的语义很明确，就是复制文件而已，而 ADD 则包含了更复杂的功能，其行为也不一定很清晰。最适合使用 ADD 的场合，就是所提及的需要自动解压缩的场合。

在 COPY 和 ADD 指令中选择的时候，可以遵循这样的原则，所有的文件复制均使用 COPY 指令，仅在需要自动解压缩的场合使用 ADD。


## 2.6 CMD 容器启动命令
Docker不是虚拟机，容器就是进程。既然是进程，那么在启动容器的时候，需要指定所运行的程序及参数。CMD 指令就是用于指定默认的容器主进程的启动命令的。
CMD 指令的格式和 RUN 相似，也是两种格式：

- shell 格式：CMD <命令>
- exec 格式：CMD ["可执行文件", "参数1", "参数2"...]
- 参数列表格式：CMD ["参数1", "参数2"...]。在指定了 ENTRYPOINT 指令后，用 CMD 指定具体的参数。

如果使用 shell 格式的话，实际的命令会被包装为 sh -c 的参数的形式进行执行。比如：

	CMD echo $HOME
在实际执行中，会将其变更为：

	CMD [ "sh", "-c", "echo $HOME" ]

注意事项——  
Docker 不是虚拟机，容器中的应用都应该以前台执行，而不是像虚拟机、物理机里面那样，用 upstart/systemd 去启动后台服务，容器内没有后台服务的概念。

一些初学者将 CMD 写为：

	CMD service nginx start

然后发现容器执行后就立即退出了。甚至在容器内去使用 systemctl 命令结果却发现根本执行不了。这就是因为没有搞明白前台、后台的概念，没有区分容器和虚拟机的差异，依旧在以传统虚拟机的角度去理解容器。

正确的做法是直接执行 nginx 可执行文件，并且要求以前台形式运行。比如：

	CMD ["nginx", "-g", "daemon off;"]

# REF
> [https://docs.docker.com/engine/reference/builder/](https://docs.docker.com/engine/reference/builder/)