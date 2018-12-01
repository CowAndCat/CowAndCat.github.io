---
layout: post
title: Flask
category: python
comments: false
---

## 1. 安装Flask

1. 先安装virtualenv
    
    这个东西，是为单个project构造独立的虚拟的开发环境，因为python不同的版本不是向后兼容的，而且不同的项目有不同的依赖，这个能避免那样的场景。

    安装：

        $ sudo pip install virtualenv

    使用：

        $ mkdir myproject
        $ cd myproject
        $ virtualenv venv
        New python executable in venv/bin/python
        Installing setuptools, pip............done.

    当你想在单个project里开展工作，执行：

        $ source venv/bin/activate

    Go back to the real world:

        $ deactivate

2. 安装Flask

    $ . venv/bin/activate
    $ pip install Flask

或者用`sudo pip install Flask`将Flask安装到全局里。

## 2. 使用Flask

main.py:

    from flask import Flask
    app = Flask(__name__)

    @app.route('/')
    def hello_world():
        return 'Hello, World!'
运行：
    
    $ export FLASK_APP=main.py
    $ flask run
     * Running on http://127.0.0.1:5000/

    或
    $ python -m flask run --host=0.0.0.0

开启调试模式：

    $ export FLASK_DEBUG=1

## 3. 实践出真理

1. 配置使用YAML：

        pip install PyYAML

        import yaml
        Conf = yaml.load(file('../conf.yml', 'r'))
        print Conf['conf']

2. python import的层级关系

        [http://www.cnblogs.com/kex1n/p/5971590.html](http://www.cnblogs.com/kex1n/p/5971590.html)




