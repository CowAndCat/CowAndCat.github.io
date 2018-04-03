---
layout: post
title: Sublime Text 编辑 Python
category: python
comments: false
---

## 1. 环境安装

自行百度，谢谢。

## 2. 编辑py文件的时候代码都有个白框

是不是安装了 SublimeLinter？

这是用来在写代码时做代码检查的。可以在包管理器中安装。写Python程序的话，它还会帮你查代码是否符合PEP8的要求。有问题有代码会出现白框，点击时底下的状态栏会提示出什么问题。

建议再装一个 Python PEP8 Autoformat

这是用来按PEP8自动格式化代码的。可以在包管理器中安装。如果以前写程序不留意的话，用SublimeLinter一查，满屏都是白框框，只要装上这个包，按**ctrl+shift+r**，代码就会按PEP8要求自动格式化了，一屏的白框几乎都消失了。