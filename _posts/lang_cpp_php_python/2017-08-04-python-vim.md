---
layout: post
title: Vim Conf and Config It for Python (Mac)
category: python
comments: false
---
## 1. 在Vim里配置python的自动补全功能

1. 下载补全工具pydiction:

        cd ~/.vim/bundle 
        git clone https://github.com/rkulla/pydiction.git 

2. 将.vim文件放在合适的位置：
    
        cd ~/.vim
        mv bundle/pydiction/after .

3. 在vimrc里增加配置：
        
        vim ~/.vimrc
        增加内容：
            filetype plugin on 
            let g:pydiction_location = '~/.vim/bundle/pydiction/complete-dict'
            let g:pydiction_menu_height = 5

4. 使用

    编辑python文件，在想要补全的时候按Tab键。

## 2. Vim 常用配置

        set tabstop=4   # 设置（软）制表符宽度为4
        set softtabstop=4
        set shiftwidth=4 # 设置缩进的空格数为4
        set autoindent   # 自动缩进，使用 noautoindent 取消设置
        set cindent      # C/C++语言的自动缩进
        set cinoptions={0,1s,t0,n-2,p2s,(03s,=.5s,>1s,=1s,:1s # 设置C/C++语言的具体缩进方式
        set expandtab
        set nu          # 显示行号

## 3. Vim 配色

1. 配色方案  
    查看现有配色

        ls -l  /usr/share/vim/vim73/colors/

        -rw-r--r--  1 root      wheel                  2311  9  1  2009 README.txt
        -rw-r--r--  1 root      wheel                  2476  9  1  2009 blue.vim
        -rw-r--r--  1 root      wheel                  2990  9  1  2009 darkblue.vim
        -rw-r--r--  1 root      wheel                   548  9  1  2009 default.vim
        -rw-r--r--  1 root      wheel                  2399  9  1  2009 delek.vim
        -rw-r--r--  1 root      wheel                  2812  9  1  2009 desert.vim
        -rw-r--r--  1 root      wheel                  1666  9  1  2009 elflord.vim
        -rw-r--r--  1 root      wheel                  2476  9  1  2009 evening.vim
        -rw-r--r--@ 1 maofagui  INTERNAL\Domain Users  8791  8  4 11:17 jellybeans.vim

    如果我们想要将配色方案改为evening，那么我们只需要在.vimrc中增加一行```colorscheme evening```即可。

2. 下载配色  
    如果觉得配色方案太少，可以从外部下载配色方案，然后将.vim的文件放入colors/目录下，最后更改.vimrc即可生效。

    上述 jellybeans.vim 是笔者自行下载的，可以作为一个配色推荐。

    几个配色网址：

    + [vim的可视化在线配色器](http://bytefluent.com/vivify/)
    + [solarized](http://ethanschoonover.com/solarized)
    + [http://vimcolorschemetest.googlecode.com/svn/colors/](http://vimcolorschemetest.googlecode.com/svn/colors/)

## 4. 语法高亮

1. 打开vimrc，添加以下语句来使得语法高亮显示：

        syntax on

2. 如果此时语法还是没有高亮显示，那么在~/.bash_profile(或在/etc目录下的profile文件)中添加以下语句：

        export TERM=xterm-color

