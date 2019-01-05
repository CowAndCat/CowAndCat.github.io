---
layout: post
title: find 命令学习
category: linux
comments: false
---

TBC


## 一、find命令查找包含指定内容的文件

find / | xargs grep function 查找系统根目录下面的所有文件的内容中包含有function字符串的文件列表。

    find .|xargs grep x
    find . -exec grep x{} \;

    find / -name "httpd.conf"

    find bce-mct-remote-process-platform.error.log.20181229* | xargs grep Broken

为什么会这样能，因为通道命令符是把上一部的结果传递给下一步来处理，在 find . |grep x中虽然看似和find .|xargs grep x差不多，但是实际上还是有区别的。应为find .得到的结果是一串文件名集合，如果直接传递给grep的话，grep会把这些文件名看作一些无意义的字符串来处理。但是传递给xargs，他会把他当作一个有意义的文件来处理。

## REF
> [https://www.cnblogs.com/xiaoleiel/p/8349495.html](https://www.cnblogs.com/xiaoleiel/p/8349495.html)