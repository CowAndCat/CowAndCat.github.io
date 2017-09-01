---
layout: post
title: Shell Script编写
category: linux
comments: false
---

# 1. 从一个例子说起

    [dmtsai@study bin]$ vim hello.sh
    #!/bin/bash
    # Program:
    #       This program shows "Hello World!" in your screen.
    # History:
    # 2015/07/16    VBird   First release
    PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:~/bin
    export PATH
    echo -e "Hello World! \a \n"
    exit 0

第一行:  #!/bin/bash 表示这个文档中的语法是使用bash这个shell的语法。  
下面几行是注释，再下面的几行也很简单，不赘述。

最后一个exit 0返回的结果会传到 $? 里面。

shell script 的执行，至少需要有r 的权限，若需要直接指令下达，则需要拥有r 与x 的权限。 

若需要进行判断式，可使用test 或中括号( [] ) 来处理。

# 2. 再来一个小测试
撰写一个script ，让使用者输入：1. first name与2. last name，最后并且在萤幕上显示：『Your full name is: 』

答案：
    
    #!/bin/bash
    # Program:
    # User inputs his first name and last name. Program shows his full name.
    # History:
    # 2015/07/16 VBird First release
    PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:~/bin
    export PATH

    read -p "Please input your first name: " firstname       #提示使用者输入 
    read -p "Please input your last name: " lastname         #提示使用者输入 
    echo -e "\nYour full name is: ${firstname} ${ lastname}" #结果由萤幕输出

# 3. 执行脚本的差异

1. 利用直接执行的方式来执行script，如 sh XX.sh 或 ./XX.sh, 会另开子程序执行
2. source XX.sh 会在该父线程里执行。

# 4. 脚本参数

- $# ：代表后接的参数『个数』；
- "$@" ：代表『 "$1" "$2" "$3" "$4" 』之意，每个变数是独立的(用双引号括起来)；
- "$*" ：代表『 "$1 c $2 c $3 c $4" 』，其中c为分隔字元，预设为空白键，所以本例中代表『 "$1 $2 $3 $4" 』之意。
- shift：造成参数变数号码偏移 默认移1位(使用后第一个参数就没了），后面可以接数字。

一个栗子：

    [dmtsai@study bin]$ vim how_paras.sh 
    #!/bin/bash
    # Program:
    # Program shows the script name, parameters...
    # History:
    # 2015/07/16 VBird First release
    PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:~/bin
    export PATH

    echo "The script name is ==> ${0}"
    echo "Total parameter number is ==> $#"
    [ "$#" -lt 2 ] && echo "The number of parameter is less than 2. Stop here." && exit 0
    echo "Your whole parameter is ==> '$@'"
    echo "The 1st parameter ==> ${1}"
    echo "The 2nd parameter ==> ${2}"

脚本运行结果：

    [dmtsai@study bin]$ sh how_paras.sh theone haha quot 
    The script name is ==> how_paras.sh        <==档名 
    Total parameter number is ==> 3                   <==果然有三个参数 
    Your whole parameter is == > 'theone haha quot' <==参数的内容全部 
    The 1st parameter ==> theone              <==第一个参数 
    The 2nd parameter ==> haha                <==第二个参数

# 5. 条件判断
## 5.1 利用if .... then

- 单层、简单条件判断式

        if [条件判断式]; then
            当条件判断式成立时，可以进行的指令工作内容；
        fi    <==将if反过来写，就成为fi啦！结束if之意！

- 多重、复杂条件判断式
    
        #多个条件判断(if ... elif ... elif ... else)分多种不同情况执行
        if [条件判断式一]; then
            当条件判断式一成立时，可以进行的指令工作内容；
        elif [条件判断式二]; then
            当条件判断式二成立时，可以进行的指令工作内容；
        else
            当条件判断式一与二均不成立时，可以进行的指令工作内容；
        fi

## 5.2 利用case ..... esac 判断

    case $变数名称in    <==关键字为case ，还有变数前有钱字号 
      "第一个变数内容" )    <==每个变数内容建议用双引号括起来，关键字则为小括号)
        程式段
        ;;             <==每个类别结尾使用两个连续的分号来处理！
      "第二个变数内容" )
        程式段
        ;; 
      * )                   <==最后一个变数内容都会用*来代表所有其他值
        不包含第一个变数内容与第二个变数内容的其他程式执行段
        exit 1
        ;; 
    esac                   <==最终的case结尾！『反过来写』思考一下！

一个栗子：

    case ${1} in
      "one")
        echo "Your choice is ONE"
        ;;
      "two")
        echo "Your choice is TWO"
        ;;
      "three")
        echo "Your choice is THREE"
        ;;
      *)
        echo "Usage ${0} {one|two|three}"
        ;;
    esac

## 5.3 loop

常见的while:

    while [ condition ]   <==中括号内的状态就是判断式 
    do             <==do是回圈的开始！
        程式段落
    done           <==done是回圈的结束

与while相反的until：（condition为真的时候终止loop）

    until [ condition ]
    do
        程式段落
    done

for：

    for var in [con1 con2 con3] ...
     do
        程式段
    done

或

    for ((初始值;限制值;执行步阶))
    do
        程式段
    done

一个for栗子：

    for animal in dog cat elephant
    do
        echo "There are ${animal}s.... "
    done

    for num in $(seq 1 100) 
    balabala
    # 遍历1~100

# 6. shell script 的追踪与debug

script也是可以调试的！！

    [dmtsai@study ~]$ sh [-nvx] scripts.sh 
    选项与参数：
    -n ：不要执行script，仅查询语法的问题；
    -v ：再执行sccript 前，先将scripts 的内容输出到萤幕上；
    -x ：将使用到的script 内容显示到萤幕上，这是很有用的参数！

    范例一：测试dir_perm.sh有无语法的问题？
    [dmtsai@study ~]$ sh -n dir_perm.sh 
    #若语法没有问题，则不会显示任何资讯！

    范例二：将show_animal.sh的执行过程全部列出来～ 
    [dmtsai@study ~]$ sh -x show_animal.sh 
    + PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local /bin:/usr/local/sbin:/root/bin
    + export PATH
    + for animal in dog cat elephant
    + echo 'There are dogs.... '
    There are dogs....
    + for animal in dog cat elephant
    + echo 'There are cats.... '
    There are cats....
    + for animal in dog cat elephant
    + echo 'There are elephants.... '
    There are elephants....

可使用sh -x script.sh 来进行程式的debug




