MFG's Blog
==========

基于 [StrayBirds](http://minixalpha.github.io/StrayBirds/)  搭建的极简博客，所有操作都可以直接通过浏览器完成。

本博客模板仅包含一个皮肤：

[kunka](https://github.com/pizn/kunka), Licence: MIT, author: [zhanxin.info](http://www.zhanxin.info/)

## 教程

### 使用方法

1. 注册 GitHub，得到用户名，例如 minixbeta
2. 到 [StrayBirds](https://github.com/minixalpha/StrayBirds) 页面，单击右上
角的 Fork
3. 到你 Fork 后的项目中，将 `_config.yml` 中的 username 修改为你的用户名 minixbeta
4. 访问你的博客 http://minixbeta.github.io/StrayBirds/

  **注意如果你是第一次使用 GitHub Pages，可能不会马上生效，等一段时间即可**

   **按照配置中说的方法修改项目名称可能会加快这一进程**

### 配置

* 修改评论系统用户名

    这里的评论系统使用的是 [Disqus](https://disqus.com/)，如果你想在这份博客模板中使用，需要先去注册一下，然后得到一个用户名，例如 minixalpha。然后在 `_config.yml` 中将 disqusname 修改为 minixalpha。

    **千万注意: 如果你开启评论系统一定要修改这个值，不然就评论到我的评论系统中去了**

### 添加文章

在 `_post` 目录下添加形如 `2015-08-07-title.md` 的文章，用 markdown 格式
撰写博客。

例如：

```
---
layout: post
title: Java 中的并发
comments: true
category: 技术
---


## 如何创建一个线程

按 Java 语言规范中的说法，创建线程只有一种方式，就是创建一个 Thread 对象。而从 HotSpot 虚拟机的角度看，创建一个虚拟机线程
有两种方式，一种是创建 Thread 对象，另一种是创建 一个本地线程，加入到虚拟机线程中。

...

```

其中 `layout` 表示布局，不用改变，`title` 表示文章题目，`comments` 表示是否要开户评论。

## 其他

如果想本地调试生成的网页，可以搭建jekyll运行环境. 参考：
[http://www.2cto.com/os/201411/351818.html](http://www.2cto.com/os/201411/351818.html)

如果遇到错误，可以参考：
[https://ruby-china.org/topics/29323](https://ruby-china.org/topics/29323)
[http://zyzhang.github.io/blog/2012/08/31/highlight-with-Jekyll-and-Pygments/](http://zyzhang.github.io/blog/2012/08/31/highlight-with-Jekyll-and-Pygments/)
[https://gems.ruby-china.org/](https://gems.ruby-china.org/)

其他插件：
gem install redcarpet jekyll-paginate
(貌似redcarpet在github的页面不再受到支持，考虑用替代品：kramdown)
