---
layout: post
title: Python 代码混淆、编译、打包、运行
category: python
comments: false
---

像Python这种解释性的语言，要想私有化部署的同时又保护好源码，就像是对于鱼和熊掌的追求。

虽然做不到尽善尽美，但是对代码进行混淆，增加一点破解的难度，或许能规避一些泄露的风险。

# 一、Python 工程的编译、合并、打包、发布

确保要发布的包(demo)的根目录中有__main__.py文件，这个是程序执行入口。

编译

    python3 -O -m compileall demo

批量改名.pyc文件

    find . -name '*.pyc' -exec rename 's/.cpython-35.opt-1//' {} \;

移动.pyc文件

    find . -name '*.pyc' -execdir mv {} .. \;

清理.py文件

    find . -name '*.py' -type f -print -exec rm {} \;

清理__pycache__文件夹

    find . -name '__pycache__' -exec rmdir {} \;

打包成zip

    zip -r pub.zip ./demo/*

运行时只要将zip文件作为参数即可

    python3 pub.zip
 

最终整合脚本

    cd $1
    python3 -O -m compileall .
    find . -name '*.pyc' -exec rename 's/.cpython-35.opt-1//' {} \;
    find . -name '*.pyc' -execdir mv {} .. \;
    find . -name '*.py' -type f -print -exec rm {} \;
    find . -name '__pycache__' -exec rmdir {} \;
    zip -r ../$1.zip ./*

调用方式 

    chmod +x pycompile.sh
    ./pycompile.sh demo

# 二、Python 混淆


对于在变量和函数名上的混淆有点小儿科，而对于跨文件的类名的混淆又太容易实现。

所以对于混淆程度的取舍，要视工程的规模而定。

- 如果只有单个文件，那可以多种混淆合用，并最后用base64和压缩算法对代码做终极保护。
- 如果有很多关联的工程，那就只能试探性地增加混淆算法，直到编译不通过打止。

## 2.1 混淆工具

混淆工具：

pyminifier: https://github.com/haluomao/pyminifier

在原来的工具 [pyminifier](https://github.com/liftoff/pyminifier)上修复了几个bug。   

- 对`__init__`空文件的大小计算时出现除以0的情况
- 使用pyz功能时，文件参数不正确

安装： 

    pip install git+git://github.com/haluomao/pyminifier.git@master

python3 安装

    pip3 install git+git://github.com/haluomao/pyminifier.git@master

或者clone下来，自行安装

    git clone https://github.com/haluomao/pyminifier.git
    cd pyminifier
    python setup.py install

使用例子：
    
    pyminifier -O test.py
    pyminifier --pyz=test.pyz test.py


## 2.2 源码变更

不同的配置对于源码的要求不同，以下是笔者踩过的坑。

- --obfuscate-builtins： 对内置变量进行混淆，所以源码中不能出现以file，reload等关键字命名的变量。
- --obfuscate-variables：对变量进行改名混淆。嵌套循环的时候，index变量都被重命名为i（好傻），所以在上层循环的前面，把这层循环的index变量设置成None。
- --nonlatin：用unicode字符串代替变量，仅在python3中支持。很可能导致编译不通过。
- --obfuscate-classes：对类名混淆，跨文件支持很差。
- --obfuscate-functions：对函数名混淆，支持也差。
- --pyz：能自动打包main文件所依赖的模块，但是只会做minify，不会做混淆。而且打包的.pyz里各文件依旧是源码。

## 其他混淆想法

> [通过字节码混淆来保护Python代码](https://blog.csdn.net/ir0nf1st/article/details/61650984)

# 三、最终的编译混淆脚本：

结合混淆、编译和打包，尝试出以下发布脚本。

主要的思路：创建一个工作目录tmp，然后在此目录下混淆、编译python代码，完成后把内容打包成pyc文件，再将pyc文件和其他配置文件移动到dist，发布dist即可。

    #!/bin/bash
    #
    # Author: Felix
    # Brief:
    #   To obfuscate and compile *.py
    #set -o

    if [[ -d "tmp" ]];then
        rm -rf tmp
    fi
    mkdir tmp
    mkdir tmp/src 2>&1
    cp -r src/* tmp/src/
    cd tmp

    find ./src -name '*.pyc' -type f -print -exec rm {} \;
    #--nonlatin --replacement-length=5
    #--obfuscate-classes --obfuscate-functions --obfuscate-variables --obfuscate-import-methods --obfuscate-builtins
    find ./src -name '*.py' -exec pyminifier -o {} --obfuscate-builtins --obfuscate-import-methods --obfuscate-variables --replacement-length=5 {} \;
    python3 -O -m compileall -x 'excluded*' src
    find ./src -name '*.pyc' -exec rename 's/.cpython-37.opt-1//g' {} \;
    find ./src -name '*.pyc' -execdir mv {} .. \;
    find ./src -name '*.py' -type f -print -exec rm {} \;
    find ./src -name '__pycache__' -exec rmdir {} \;
    cd src
    zip -r ../../dist/$1.pyc . -x "*/\.*"-x "venv/*" -x "dist/*" -x "log/*" -x "scripts/*" -x "*.py" -x "*.txt" -x "*.sh" -x "*.md"
    cd ../..

    if [[ -d "dist/log" ]]; then
        rm -rf dist/log
    fi
    if [[ -d "dist/conf" ]]; then
        rm -rf dist/conf
    fi
    if [[ -d "dist/datasets" ]]; then
        rm -rf dist/datasets
    fi
    mkdir dist/log
    mkdir dist/conf
    mkdir dist/datasets
    cp -r conf/* dist/conf/
    cp -r datasets/*.md5 dist/datasets/












