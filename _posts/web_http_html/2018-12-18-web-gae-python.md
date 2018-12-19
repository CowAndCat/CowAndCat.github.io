---
layout: post
title: 用GAE统计静态网页的访问量
category: web
comments: true
---

## 一、搭建GAE编程环境

首先下载Google App Engine的SDK： [Download and Install the SDK for App Engine](https://cloud.google.com/appengine/downloads?csw=1) 选择python

    sh ./google-cloud-sdk/install.sh
    ./bin/gcloud init --skip-diagnostics
    # 登录选择Y,不然后面还是要登录
    ./google-cloud-sdk/bin/gcloud components install app-engine-python

安装完毕！

遇到问题： gcloud auth login 浏览器登录后没有回调成功。
解决：不通过浏览器登录。
    
    gcloud auth login ACCOUNT --no-launch-browser

需要输入验证码。。

仔细看输出，发现仍旧要复制那一长串的URL，浏览器登录验证，到第三步会显示出verification code。

输入后仍旧不行！可能公司把某些网站给墙了（ping cloud.google.com）。

各位读者老爷，下面的内容也别看了，未完待续。

## 二、快速开始

首先到项目页面创建一个项目，进入
https://console.cloud.google.com/iam-admin/projects 选择『创建项目』

比如，我写的项目名称为 ViewStatistics， id为viewstatistics。

### 2.1 编写项目

好，回到电脑上，用PyCharm创建一个Python项目，并创建好下面的文件和目录：

    - app.yaml: Configure the settings of your App Engine application
    - main.py: Write the content of your application
    - static: Directory to store your static files
        - style.css: Basic stylesheet that formats the look and feel of your template files
    - templates: Directory for all of your HTML templates
        - form.html: HTML template to display your form
        - submitted_form.html: HTML template to display the content of a submitted form
    - appengine_config.py
        from google.appengine.ext import vendor
        # Add any libraries installed in the "lib" folder.
        vendor.add('lib')
    - requirements.txt
        Flask==1.0.2
        Werkzeug<0.13.0,>=0.12.0

上述文件都可以github上下载到： https://github.com/GoogleCloudPlatform/python-docs-samples/tree/master/appengine/standard/flask/tutorial

在PyCharm里点击terminal进入venv，安装依赖：

    pip install -t lib -r requirements.txt

然后参照github继续补全内容。 注意venv里要用python2.7.

### 2.2 测试应用

    dev_appserver.py app.yaml

启动后访问：http://localhost:8080/form

发现报错。。

    ImportError: No module named _ssl

解决：在app.yaml里增加配置

    libraries:
    - name: ssl
      version: latest

瞎填一通，提交页面后能正常跳转：

![gae](/images/201812/gae-submitted.png)

### 2.3 发布它

    gcloud app deploy --project viewstatistics -v 0-1

可选参数：

- --project： Include the --project flag to specify an alternate GCP Console project ID to what you initialized as the default in the gcloud tool. Example: --project [YOUR_PROJECT_ID]
- -v: Include the -v flag to specify a version ID, otherwise one is generated for you. Example: -v [YOUR_VERSION_ID] 注意版本名只能用lowercase letters, digits, and hyphens（-）

## TODO

等哪天gcloud auth 能通过了，再继续研究。

## REF
> [对 Cloud SDK 工具授权](https://cloud.google.com/sdk/docs/authorizing)  
> [gcloud auth login](https://cloud.google.com/sdk/gcloud/reference/auth/login)  
> [Getting Started with Flask on App Engine Standard Environment](https://cloud.google.com/appengine/docs/standard/python/getting-started/python-standard-env)  
> [实践 | 用SAE搭建Python应用，实现简单的网页访问量统计](https://www.jianshu.com/p/a0ffe6de9617)
