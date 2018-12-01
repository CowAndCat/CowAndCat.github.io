---
layout: post
title: Virtualenv搭建独立的python运行环境
category: python
comments: false
---

一开始是因为使用PyCharm IDE才接触到它。
后来因为制作Docker的需求，才安装它，虽然最后没能解决我的需求，但还是为它写一篇文章吧。

## 一、virtualenv

作为Python三大神器之一（其他两位是pip和fabric），可以用来快速搭建一个干净独立的python环境。

### 1.1 安装和使用

安装：
	
	pip install virtualenv
  
测试安装:

	virtualenv --version

为一个工程项目搭建一个虚拟环境:

	cd my_project
	virtualenv my_project_env

如果系统存在多个python解释器，可以选择指定一个Python解释器（比如``python2.7``），没有指定则由系统默认的解释器来搭建： 

	virtualenv -p /usr/bin/python2.7 my_project_env

要开始使用虚拟环境，需要激活它：

	source my_project_env/bin/activate

这时会在命令行前面出现一个类似 (venv)	的东西，说明现在的命令都会优先使用venv下的环境（主要指python）。

停用虚拟环境：

	deactivate

例如要快速构建一套独立的、安装有依赖的python环境，可以执行下面这些命令：

	cd workdir
	virtualenv -p python3 venv
	source venv/bin/activate
	pip3 install -r requirements.txt
	deactivate

venv文件夹里的内容就是环境了。但是复制venv到另外一个同样的系统，可能并不能生效，因为virtualenv会复用原系统的环境依赖，比如.so文件，这类文件不会复制到venv下。

## 二、真正独立的python环境

将python安装到想要的目录下，然后使用该目录下的环境安装其他依赖，这个时候，可以得到一个真正的可移动的Python环境。

在下载好Python源码的目录（这里是/root/felix/matcher），执行如下命令：

	tar -zxvf Python-3.7.1.tar.gz
	cd Python-3.7.1
	./configure --prefix=/root/felix/matcher/pEnv
	make -j3
	make install
	cd ..

安装依赖到独立环境：

	./pEnv/bin/pip3 install -r requirements.txt

如果没有网络，也可以安装离线的whl：

	./pEnv/bin/pip3 install Flask-1.0.2-py2.py3-none-any.whl
	...

之后执行py程序可以用命令：

	./pEnv/bin/python3 matcher.py

移动整个matcher文件夹，上述命令也是正常的。



