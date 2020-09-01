---
layout: post
title: 中文敏感词匹配
category: NLP
comments: false
---

本文记录了笔者实现一个中文敏感词匹配微服务的思路和过程。

# 一、需求分解成卡片

TODO:

1. [P1] 中文分词器选定Jieba, 调研安装使用
2. [P1] 敏感词匹配算法（本地词包）
3. [P1] 支持自定义词包（网络格式，或DB模式）
4. [P2] 封装成Python微服务，支持Restful接口。（制定输入，输出）
5. [P3] 实现成docker模式。
6. [P1] 对词包进行加密。

ALL done on 2018-11-30.

# 二、实现细节

## 2.1 中文分词

收集的python 中文分词开源库共六款，Jieba排名靠前：[python 中文分词开源库](https://blog.csdn.net/sinat_26917383/article/details/77067515)

就选定Jieba作为本项目的分词器。

Jieba 特点：

- 支持三种分词模式：
    - 精确模式，试图将句子最精确地切开，适合文本分析；
    - 全模式，把句子中所有的可以成词的词语都扫描出来, 速度非常快，但是不能解决歧义；
    - 搜索引擎模式，在精确模式的基础上，对长词再次切分，提高召回率，适合用于搜索引擎分词。
- 支持繁体分词
- 支持自定义词典

Jieba的算法：

- 基于前缀词典实现高效的词图扫描，生成句子中汉字所有可能成词情况所构成的有向无环图 (DAG)
- 采用了动态规划查找最大概率路径, 找出基于词频的最大切分组合
- 对于未登录词，采用了基于汉字成词能力的 HMM 模型，使用了 Viterbi 算法

HMM 模型是隐马尔可夫模型（Hidden Markov Model，HMM）是统计模型，它用来描述一个含有隐含未知参数的马尔可夫过程。

Jieba还有其他的语言实现，如Java（https://github.com/huaban/jieba-analysis）、C++、Node.js等。

分词速度

- 1.5 MB / Second in Full Mode
- 400 KB / Second in Default Mode
- 测试环境: Intel(R) Core(TM) i7-2600 CPU @ 3.4GHz；《围城》.txt

### 2.1.1 Jieba 安装

先配置python 2.7环境，然后下载PyCharm作为项目的IDE。

Jieba: [https://github.com/fxsjy/jieba](https://github.com/fxsjy/jieba)

安装：sudo pip install jieba

### 2.1.2 学习使用

因为用Jieba是为了中文分词功能，所以只看这部分的教程。

#### 0.分词

- jieba.cut 方法接受三个输入参数: 需要分词的字符串；cut_all 参数用来控制是否采用全模式；HMM 参数用来控制是否使用 HMM 模型
- jieba.cut_for_search 方法接受两个参数：需要分词的字符串；是否使用 HMM 模型。该方法适合用于搜索引擎构建倒排索引的分词，粒度比较细
- 待分词的字符串可以是 unicode 或 UTF-8 字符串、GBK 字符串。注意：不建议直接输入 GBK 字符串，可能无法预料地错误解码成 UTF-8
- jieba.cut 以及 jieba.cut_for_search 返回的结构都是一个可迭代的 generator，可以使用 for 循环来获得分词后得到的每一个词语(unicode)，或者用
- jieba.lcut 以及 jieba.lcut_for_search 直接返回 list
- jieba.Tokenizer(dictionary=DEFAULT_DICT) 新建自定义分词器，可用于同时使用不同词典。jieba.dt 为默认分词器，所有全局分词相关函数都是该分词器的映射。

代码示例1：

    print('/'.join(jieba.cut('如果放到post中将出错。', HMM=False)))
    
代码示例2：

    # encoding=utf-8
    import jieba

    seg_list = jieba.cut("我来到北京清华大学", cut_all=True)
    print("Full Mode: " + "/ ".join(seg_list))  # 全模式

    seg_list = jieba.cut("我来到北京清华大学", cut_all=False)
    print("Default Mode: " + "/ ".join(seg_list))  # 精确模式

    seg_list = jieba.cut("他来到了网易杭研大厦")  # 默认是精确模式
    print(", ".join(seg_list))

    seg_list = jieba.cut_for_search("小明硕士毕业于中国科学院计算所，后在日本京都大学深造")  # 搜索引擎模式
    print(", ".join(seg_list))

#### 1.Jieba并行分词介绍（备用）

```
并行分词

原理：将目标文本按行分隔后，把各行文本分配到多个 Python 进程并行分词，然后归并结果，从而获得分词速度的可观提升

基于 python 自带的 multiprocessing 模块，目前暂不支持 Windows

用法：

jieba.enable_parallel(4) # 开启并行分词模式，参数为并行进程数
jieba.disable_parallel() # 关闭并行分词模式
例子：https://github.com/fxsjy/jieba/blob/master/test/parallel/test_file.py

实验结果：在 4 核 3.4GHz Linux 机器上，对金庸全集进行精确分词，获得了 1MB/s 的速度，是单进程版的 3.3 倍。

注意：并行分词仅支持默认分词器 jieba.dt 和 jieba.posseg.dt。
```

####  2.命令行分词

```
使用示例：python -m jieba news.txt > cut_result.txt

命令行选项（翻译）：

    使用: python -m jieba [options] filename

    结巴命令行界面。

    固定参数:
      filename              输入文件

    可选参数:
      -h, --help            显示此帮助信息并退出
      -d [DELIM], --delimiter [DELIM]
                            使用 DELIM 分隔词语，而不是用默认的' / '。
                            若不指定 DELIM，则使用一个空格分隔。
      -p [DELIM], --pos [DELIM]
                            启用词性标注；如果指定 DELIM，词语和词性之间
                            用它分隔，否则用 _ 分隔
      -D DICT, --dict DICT  使用 DICT 代替默认词典
      -u USER_DICT, --user-dict USER_DICT
                            使用 USER_DICT 作为附加词典，与默认词典或自定义词典配合使用
      -a, --cut-all         全模式分词（不支持词性标注）
      -n, --no-hmm          不使用隐含马尔可夫模型
      -q, --quiet           不输出载入信息到 STDERR
      -V, --version         显示版本信息并退出

    如果没有指定文件名，则使用标准输入。

--help 选项输出：
$> python -m jieba --help
```
####  方案选择

最终采用search模式来分词。

#### 3.词典  

默认的词典放在jieba/dict.txt,其他词典有：

- 占用内存较小的词典文件 https://github.com/fxsjy/jieba/raw/master/extra_dict/dict.txt.small
- 支持繁体分词更好的词典文件 https://github.com/fxsjy/jieba/raw/master/extra_dict/dict.txt.big

下载你所需要的词典，然后覆盖 jieba/dict.txt 即可；或者用 jieba.set_dictionary('data/dict.txt.big')

#### 4.被误分的词解决办法-强制调高词频

jieba.add_word('台中') 或者 jieba.suggest_freq('台中', True)

## 2.2 匹配算法

算法很简单：集合求交集.

## 2.3 Web服务

基于Flask实现Web服务：[《Flask》](/python/2017/08/04/python-flask.html)

## 2.4 Docker实现
基础知识请移步站内文章：

- [《Docker 是什么》](/docker/2018/11/28/docker-intro.html)  
- [《Dockerfile定制镜像》](/docker/2018/11/29/docker-dockerfile.html)

Dockfile:
  
    FROM someplace/centos7:0.8

    LABEL maintainer "Felix"

    ADD matcher.tar /root/matcher/

    WORKDIR /root/matcher
    ENV LD_LIBRARY_PATH=/root/matcher:$LD_LIBRARY_PATH
    EXPOSE 5001
    CMD ["sh", "start.sh"]

启动脚本shart.sh内容：

    /root/matcher/pEnv/bin/python3 matcher.pyc

重点是matcher.tar内容，笔者将原来的matcher项目进行代码混淆编译打包后，将其依赖的Python3环境、Flask等依赖包全部都安装在pEnv文件夹下，孤儿启动的时候，只需要执行：

    pEnv/bin/python3 matcher.pyc

然后将pEnv和matcher.pyc一起打包成tar，方便了Docker build的过程。

## 2.5 词包加密

采用md5+salt的方式来保证词包内容不被泄露。

在匹配的时候，对分词得到的关键字进行同样的加密，然后对加密文本集合求交集。返回结果时，通过明文和密文的映射关系还原匹配到的词包内容。

其中salt的内容会加在Python代码里，故而对Python项目进行了混淆编译，来保证salt的安全。

参见：[《Python 代码混淆、编译、打包、运行》](/python/2018/11/22/python-project.html)

## 2.6 英文分词问题

jieba英文空格分词问题：https://segmentfault.com/q/1010000016011808

