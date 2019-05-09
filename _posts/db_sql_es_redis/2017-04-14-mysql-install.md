---
layout: post
title: mysql 安装
category: mysql
comments: false
---
# 1. centos yum install

    $ wget http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
    $ yum localinstall mysql-community-release-el7-5.noarch.rpm
    $ yum repolist enabled | grep "mysql.*-community.*"

    $ yum install mysql-community-server 
    $ whereis mysql

    $ systemctl start mysqld

## 1.1 另一种方法

    $ wget https://cdn.mysql.com//Downloads/MySQL-5.6/MySQL-server-5.6.43-1.el7.x86_64.rpm
    $ yum -y install MySQL-server-5.6.43-1.el7.x86_64.rpm
    $ wget https://cdn.mysql.com//Downloads/MySQL-5.6/MySQL-client-5.6.43-1.el7.x86_64.rpm
    $ yum -y install MySQL-client-5.6.43-1.el7.x86_64.rpm

## 1.2 卸载

    rpm -qa|grep -i mysql
    yum remove mysql mysql-server mariadb-libs-1:5.5.60-1.el7_5.x86_64 mysql-community-release-el7-5.noarch
    rm -rf /var/lib/mysql
    rm /etc/my.cnf

## 1.3 配置my.cnf

启动：

    /etc/init.d/mysql start

用命令查看my.cnf路径：

    $ mysqld --verbose --help |grep -A 1 'Default options'
    Default options are read from the following files in the given order:
    /etc/my.cnf /etc/mysql/my.cnf /usr/etc/my.cnf ~/.my.cnf

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
如果 MySQL 正在运行，首先杀之：

    /etc/init.d/mysql stop
    或
    killall -TERM mysqld 

启动 MySQL ：`mysqld_safe --user=mysql --skip-grant-tables --skip-networking &`
就可以不需要密码就进入 MySQL 了：`mysql -u root mysql`
然后就是

    >use mysql
    >update user set password=password("new_pass") where user="root";
    >flush privileges;
    >quit
    $ /etc/init.d/mysql restart
    $ mysql -uroot -p
    mysql> SET PASSWORD = PASSWORD('123456');

重新kill MySQL ，用正常方法启动 MySQL.  

    GRANT USAGE ON *.* TO 'exchanger'@'localhost' IDENTIFIED BY '123456' WITH GRANT OPTION; 
    GRANT ALL PRIVILEGES ON * . * TO 'exchanger'@'%';

# 4. 常见错误

## 4.1 Please read "Security" section of the manual to find out how to run mysqld as root!
因为MySQL为了安全，不希望root用户直接启动mysql。

与使用root用户启动mysqld相比，更好的方法是使用一个普通的、没有高级权限的用户帐户允许mysqld，例如创建一个名为mysql的用户帐户来专门管理MySQL。使用其帐启动MySQL的方法是在mysqld命令后面加上一个用户选项，这个用户属于mysqld用户组并且位于my.cnf配置文件中。例如在创建mysql帐户后，可以将下面的内容添加到my.cnf文件中：

    [mysqld]
    user=mysql

或者用mysqld_safe启动：

    mysqld_safe --user=mysql &

或者强制用root账户启动：

    mysqld --user=root &

## 4.2 ERROR 1820 (HY000): You must SET PASSWORD before executing this statement

    mysql>  SET PASSWORD = PASSWORD('123456');

## REF
> [mysql 错误集锦](https://blog.csdn.net/lyj1101066558/article/details/50668111)
