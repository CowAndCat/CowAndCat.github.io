---
layout: post
title: MySQL主从监控和报警
category: mysql
comments: false
---

## 一、监控方法

### 1.1 可以借助pt里的工具

参考[percona toolkit](/mysql/2019/02/13/mysql-tool-percona.html)

pt-table-checksum和pt-heartbeat工具。

### 1.2 通过SQL语句

判断Mysql主从是否正常，可通过主从上面的SQL和IO线程都为yes状态判断（通过awk取值，grep过滤和统计yes的个数，如果为2则为正常值）

    shell> mysql -u root -p123456 -e "show slave status\G" | grep "Running" |awk "{print $2}" | grep -c "Yes" 
    2

然后可以通过这条语句来进行监督了。

另一种方法：

    [root@slave ~]# egrep "_Running|Behind_Master" slave.log | awk ‘{print $NF}‘
    Yes
    Yes
    0

slave.log是`show slave status\G;`输出的内容。

开发一个守护进程脚本每30秒实现检测一次。如果同步出现如下错误号(1158,1159,1008,1007,1062)，跳过错误:

    #!/bin/bash
    #version 1.0
    mysql_cmd="mysql -u root -proot"
    errorno=(1158 1159 1008 1007 1062)
    while true
    do
      array=($($mysql_cmd -e "show slave status\G"|egrep ‘_Running|Behind_Master|Last_SQL_Errno‘|awk ‘{print $NF}‘))
      if [ "${array[0]}" == "Yes" -a "${array[1]}" == "Yes" -a "${array[2]}" == "0" ]
      then
        echo "MySQL is slave is ok"
      else
          for ((i=0;i<${#errorno[*]};i++))
          do
            if [ "${array[3]}" = "${errorno[$i]}" ];then
            $mysql_cmd -e "stop slave &&set global sql_slave_skip_counter=1;start slave;"
            fi
          done
          char="MySQL slave is not ok"
          echo "$char"
          echo "$char"|mail -s "$char" xxxx@qq.com
          break
      fi
      sleep 30
    done

## REF
> [带你了解zabbix如何监控mysql主从到报警触发](https://www.cnblogs.com/bixiaoyu/p/7337116.html)
> [Linux上MySQL主从同步监控脚本实现](https://blog.csdn.net/wetsion/article/details/80342559)  
> [zabbix监控mysql主从同步](http://blog.51cto.com/536410/2153230)  
> [mysql主从延迟监控](https://blog.csdn.net/jx_jy/article/details/80165656)