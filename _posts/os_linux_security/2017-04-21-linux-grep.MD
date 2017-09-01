---
layout: post
title: grep
category: linux
comments: false
---

# 1. grep

grep (global search regular expression(RE) and print out the line,全面搜索正则表达式并把行打印出来)是一种强大的文本搜索工具，它能使用正则表达式搜索文本，并把匹配的行打印出来。

命令语法：

    [root@www ~]# grep [-acinv] [--color=auto] '搜寻字符串' filename
    选项与参数：
    -a ：将 binary 文件以 text 文件的方式搜寻数据
    -c ：计算找到 '搜寻字符串' 的次数
    -i ：忽略大小写的不同，所以大小写视为相同
    -n ：顺便输出行号
    -v ：反向选择，亦即显示出没有 '搜寻字符串' 内容的那一行！
    -s ：不显示不存在或无匹配文本的错误信息。
    -h ：查询多文件时不显示文件名。
    --color=auto ：可以将找到的关键词部分加上颜色的显示喔

常用命令：

    # grep -r ‘energywise’ *        #在当前目录及其子目录下搜索'energywise'行的文件
    # grep -l -r ‘energywise’ *     #在当前目录及其子目录下搜索'energywise'行的文件，但是不显示匹配的行，只显示匹配的文件


    [dmtsai@study ~]$ grep -vn 'the' regular_express.txt #当该行没有'the'这个字串时才显示出来

## 1.1 grep 与正则表达式

字符类的搜索：如果我想要搜寻 test 或 taste 这两个单字时，可以发现到，其实它们有共通的 't?st' 存在～这个时候，我可以这样来搜寻：

    [root@www ~]# grep -n 't[ae]st' regular_express.txt
    8:I can't finish the test.
    9:Oh! The soup taste good.

后面用''包含的内容就可以直接用正则表达式。

找出空白行：

    [root@www ~]# grep -n '^$' regular_express.txt

## 1.2 grep -e 使用条件

    grep -e EST -e WAIT txt
    #并列使用多个 -e参数可以实现或条件

-e 和 -E 的区别：

    BDSHYF00077613:output-ssp maofagui$ grep -e '345|joke' test.txt
    BDSHYF00077613:output-ssp maofagui$ grep -E '345|joke' test.txt
    345
    joke
    BDSHYF00077613:output-ssp maofagui$ grep -e '345' -e 'joke' test.txt
    345
    joke

注意用 -e 不能结合 '|' 来使用。



