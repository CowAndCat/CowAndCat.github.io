---
layout: post
title: mysql 安装
category: sql
comments: false
---
# 1. centos yum install

    $ wget http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
    $ yum localinstall mysql-community-release-el7-5.noarch.rpm
    $ yum repolist enabled | grep "mysql.*-community.*"

    $ yum install mysql-community-server 
    $ whereis mysql

    $ systemctl start  mysqld

# 2. 安装之后
## 2.1 设置root密码

    update user set password=PASSWORD("123456") where user='root';

## 2.2 创建用户

    GRANT USAGE ON *.* TO 'bidder'@'localhost' IDENTIFIED BY '123456' WITH GRANT OPTION; 

    GRANT ALL PRIVILEGES ON * . * TO ‘bidder’@‘localhost';

## 2.3 建立表
    create database dsp_task;
    grant select,insert,update,delete,create,drop on dsp_task.* to bidder@localhost;
    FLUSH PRIVILEGES;

# 3. MySQL(root用户)密码重置

分别在Windows下和Linux下重置了MYSQL的root的密码：   
　　在windows下：

1：进入cmd，停止mysql服务：Net stop mysql  
到mysql的安装路径启动mysql，在bin目录下使用mysqld-nt.exe启动，

2：执行：mysqld-nt --skip-grant-tables（窗口会一直停止）

3：然后另外打开一个命入令行窗口，执行mysql（或者直接进入Mysql Command Line Cilent），此时无需输入密码即可进入。

    >use mysql  
    >update user set password=password("新密码") where user="root";  
    >flush privileges;
    >exit

4：使用任务管理器，找到mysqld-nt的进程，结束进程!  
在重新启动mysql-nt服务，就可以用新密码登录了。

在linux下：  
如果 MySQL 正在运行，首先杀之： killall -TERM mysqld。  
启动 MySQL ：`bin/safe_mysqld --skip-grant-tables &`
　　就可以不需要密码就进入 MySQL 了。
　　然后就是

    >use mysql
    >update user set password=password("new_pass") where user="root";
    >flush privileges

重新kill MySQL ，用正常方法启动 MySQL.  

    GRANT USAGE ON *.* TO 'exchanger'@'localhost' IDENTIFIED BY '123456' WITH GRANT OPTION; 
    GRANT ALL PRIVILEGES ON * . * TO 'exchanger'@'%';

