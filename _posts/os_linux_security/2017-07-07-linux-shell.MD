---
layout: post
title: Shell 命令记录
category: linux
comments: false
---

最近LeetCode上增加了DB、Shell、OS等多个类别的问题，唉，以后的程序员笔面试涉及面越来越广了。

# 1. 输出文件的第10行

解法：

- Solution 1

        sed -n 10p file.txt

- Solution 2

        awk 'FNR == 10 {print}'  file.txt
        ## OR
        awk 'NR == 10' file.txt

- Solution 3

        tail -n+10 file.txt|head -1 

- Solution 4

        cnt=0
        while read line && [ $cnt -le 10 ]; do
          let 'cnt = cnt + 1'
          if [ $cnt -eq 10 ]; then
            echo $line
            exit 0
          fi
        done < file.txt


# 2. awk命令

awk 利用pattern来输出（过滤）文件的每一行内容。内容有点多，这里只整理一些常用的命令。

相较于sed 常常作用于一整个行的处理， awk 则比较倾向于一行当中分成数个『栏位』来处理。

- 输出长度大于n的行： 
         
         awk 'length($0) >=2' file.txt

- 倒序输出两个fields （默认是按空格来分割，如果有一行有超过2个fields,其他字段不会输出）

        awk '{print $2, $1}' file.txt
        ## 将分隔符换成逗号，加上 -F ','
        awk -F ',' '{print $2, $1}' file.txt

- 输出两个标识符中间的内容
        
        awk /1/,/10/ file.txt
        ## 输出由1和10组成的首位对包含的所有行。

- 输出每行的第一个field与前面所有行的第一个字段的累加值，以及平均值

        awk '{s += $1; print s, s/NR}' file.txt

# 3. du 命令
du -sh *

# 4. ls -lht 
能看文件夹下文件和大小。

# 5. sort 命令
sort可针对文本文件的内容，以行为单位来排序。

语 法：

    sort [-bcfMnrtk][源文件][-o 输出文件] 

参 数：

    -b   忽略每行前面开始出的空格字符。
    -c   检查文件是否已经按照顺序排序。
    -f   排序时，忽略大小写字母。
    -M   将前面3个字母依照月份的缩写进行排序。
    -n   依照数值的大小排序。
    -o<输出文件>   将排序后的结果存入指定的文件。
    -r   以相反的顺序来排序。
    -t<分隔字符>   指定排序时所用的栏位分隔字符。
    -k  选择以哪个区间进行排序。

常用命令：

    - sort FILE
    - sort -u FILE #去除重复行
    - sort -r number.txt -o number.txt #倒序并将结果写回去 （这里对同个文件操作，不能用重定向）
    - sort -n -k 2 -t ‘:’ facebook.txt
        apple:10:2.5
        orange:20:3.4
        banana:30:5.5
        pear:90:2.3
    # 按照用：分隔出的第二区间来排序。

# 6. alias 命令
能设定命令别名，例子：

    alias lm='ls -al'

在设置完毕后，在terminal输入lm就等同于输入后面的命令。

可以用来做一个memo：在~/.bash_profile 里增加一行：

    alias ipwd='echo lghyycl,yhblsqt'

以后要用到这个password的时候，只要敲ipwd就能显示出来了。

单个alias命令能查看目前有哪些命名别名。

取消用命令： unalias

# 7. 查询指令是否为Bash shell 的内建命令： type

能判断这个指令是来自于外部指令(指的是其他非bash 所提供的指令) 还是内建在bash中。

有点类似which指令。

# 8. 系统变量

用env/export/declare/set能观察环境变数与常见环境变数说明。

语法：
    
    var=balala

注意=前后都不能有空格。

使用：
    
    echo $var
    echo "$var"
    echo ${var}
    cd /lib/modules/`uname -r`/kernel  #高端用法1, 注意是反引号``
    cd /lib/modules/$(uname -r)/kernel #高端用法2

重置：

    unset var

应用：

    若你有一个常去的工作目录名称为：『/cluster/server/work/taiwan_2015/003/』，如何进行该目录的简化？

    Answer：

    [dmtsai@study ~]$ work="/cluster/server/work/taiwan_2015/003/" 
    [dmtsai@study ~]$ cd $work

# 9. PS1 提示字元的设定
    PS1='[\u@\h \W]\$ ' # PS1就厉害了。这个是命令提示字元，也就是我们常见的 [root@www ~]#或[dmtsai ~]$的设定值啦！可以更动的！

可以用man bash 去查询一下PS1的相关说明，以理解底下的一些符号意义。

    \d ：可显示出『星期月日』的日期格式，如："Mon Feb 2"
    \H ：完整的主机名称。举例来说，鸟哥的练习机为『study.centos.vbird』
    \h ：仅取主机名称在第一个小数点之前的名字，如鸟哥主机则为『study』后面省略
    \t ：显示时间，为24 小时格式的『HH:MM:SS』
    \T ：显示时间，为12 小时格式的『HH:MM:SS』
    \A ：显示时间，为24 小时格式的『HH:MM』
    \@ ：显示时间，为12 小时格式的『am/pm』样式
    \u ：目前使用者的帐号名称，如『dmtsai』；
    \v ：BASH 的版本资讯，如鸟哥的测试主机版本为4.2.46(1)-release，仅取『4.2』显示
    \w ：完整的工作目录名称，由根目录写起的目录名称。但家目录会以~ 取代；
    \W ：利用basename 函数取得工作目录名称，所以仅会列出最后一个目录名。
    \# ：下达的第几个指令。
    \$ ：提示字元，如果是root 时，提示字元为# ，否则就是$ 啰～

例子：
    
    [dmtsai@study home]$ PS1='[\u@\h \w \A #\#]\$ '
    [dmtsai@study /home 17:02 #85]$

# 10. read, array: 变数键盘读取、阵列 

read 示例：

    [dmtsai@study ~]$ read [-pt] variable 
    选项与参数：
    -p ：后面可以接提示字元！
    -t ：后面可以接等待的『秒数！』这个比较有趣～不会一直等待使用者啦！

    范例一：让使用者由键盘输入一内容，将该内容变成名为atest的变数 
    [dmtsai@study ~]$ read atest 
    This is a test           <==此时游标会等待你输入！请输入左侧文字看看 
    [dmtsai@study ~]$ echo ${atest} 
    This is a test           <==你刚刚输入的资料已经变成一个变数内容！

    范例二：提示使用者30秒内输入自己的大名，将该输入字串作为名为named的变数内容 
    [dmtsai@study ~]$ read -p "Please keyin your name: " -t 30 named 
    Please keyin your name: VBird Tsai     <==注意看，会有提示字元喔！
    [dmtsai@study ~]$ echo ${named} 
    VBird Tsai               <==输入的资料又变成一个变数的内容了！

array 示例：

    范例：设定上面提到的var[1] ～ var[3]的变数。
    [dmtsai@study ~]$ var[1]="small min" 
    [dmtsai@study ~]$ var[2]="big min" 
    [dmtsai@study ~]$ var[3]="nice min" 
    [dmtsai @study ~]$ echo "${var[1]}, ${var[2]}, ${var[3]}"
    small min, big min, nice min

# 11. ulimit:档案系统及程序的限制

ulimit可以『限制使用者的某些系统资源』的，包括可以开启的档案数量，可以使用的CPU时间，可以使用的记忆体总量等等.

几个例子：

    ulimit -a #列出目前身份的所有限制资料数值
    ulimit -f 10240 #限制使用者仅能建立10MBytes以下的容量的档案 （单位是Kbytes）


# 12. history
语法：

    [dmtsai@study ~]$ history [n] 
    [dmtsai@study ~]$ history [-c] 
    [dmtsai@study ~]$ history [-raw] histfiles 
    选项与参数：
    n ：数字，意思是『要列出最近的n 笔命令列表』的意思！
    -c ：将目前的shell 中的所有history 内容全部消除
    -a ：将目前新增的history 指令新增入histfiles 中，若没有加histfiles ，
          则预设写入~/.bash_history
    -r ：将histfiles 的内容读到目前这个shell 的history 记忆中；
    -w ：将目前的history 记忆内容写入histfiles 中！

例子：

    [dmtsai@study ~]$ history -w #立刻将目前的资料写入histfile当中

# 13. !! 能用于执行上一条指令
Grammar:

    [dmtsai@study ~]$ !number 
    [dmtsai@study ~]$ !command 
    [dmtsai@study ~]$ !! 
    选项与参数：
    number ：执行第几笔指令的意思；
    command ：由最近的指令向前搜寻『指令串开头为command』的那个指令，并执行；
    !! ：就是执行上一个指令(相当于按↑按键后，按Enter)

Examples:

    [dmtsai@study ~]$ !66   <==执行第66笔指令 
    [dmtsai@study ~]$ !!    <==执行上一个指令，本例中亦即!66 
    [dmtsai@study ~]$ !al   <==执行最近以al为开头的指令(上头列出的第67个)


# 14. /dev/null 垃圾桶黑洞装置与特殊写法

    范例：将错误的资料丢弃，萤幕上显示正确的资料 
    [dmtsai@study ~]$ find /home -name .bashrc 2> /dev/null 
    /home/dmtsai/.bashrc   <==只有stdout会显示到萤幕上， stderr被丢弃了

    范例：将指令的资料全部写入名为list的档案中 
    [dmtsai@study ~]$ find /home -name .bashrc > list 2> list   <==错误 
    [dmtsai@study ~]$ find /home - name .bashrc > list 2>&1      <==正确 
    [dmtsai@study ~]$ find /home -name .bashrc &> list          <==正确

# 15. 命令执行的判断依据： ; , &&, ||

在指令与指令中间利用分号(;) 来隔开，这样一来，分号前的指令执行完后就会立刻接着执行后面的指令了。

    [root@study ~]# sync; sync; shutdown -h now

如果两个指令之间有相依性（比如要等上一命令执行成功），而这个相依性主要判断的地方就在于前一个指令执行的结果是否正确。『若前一个指令执行的结果为正确，在Linux底下会回传一个$? = 0的值』于是可以借助&&和||来实现命令相依性。

|指令下达情况|  说明|
|---|---|
|cmd1 && cmd2 |   1.若cmd1执行完毕且正确执行($?=0)，则开始执行cmd2。   2.若cmd1执行完毕且为错误($?≠0)，则cmd2不执行。|
| cmd1 11 cmd2   | 1.若cmd1执行完毕且正确执行($?=0)，则cmd2不执行。 2.若cmd1执行完毕且为错误($?≠0)，则开始执行cmd2。|

(在表格里转义不了||，这里用11代替)

    范例：始终会建立/tmp/abc/hehe目录
    [dmtsai@study ~]$ ls /tmp/abc || mkdir /tmp/abc && touch /tmp/abc/hehe

# 16. 双向重导向： tee

tee 会同时将资料流分送到档案去与萤幕(screen)；而输出到萤幕的，其实就是stdout

    [dmtsai@study ~]$ ls -l /home | tee ~/homefile | more 
    #这个范例则是将ls的资料存一份到~/homefile ，同时萤幕也有输出讯息！

# 17. 字元转换命令： tr, col, join, paste, expand

tr 可以用来删除一段讯息当中的文字，或者是进行文字讯息的替换！   
经col -x 的处理，会将[tab] 取代成为对等的空白键！  
join:处理『两个档案当中，有"相同资料" 的那一行，才将他加在一起』  
paste:直接『将两行贴在一起，且中间以[tab]键隔开』 
expand: 将[tab] 按键转成空白键

# 18. 分割命令： split
可以帮你将一个大档案，依据档案大小或行数来分割，就可以将大档案分割成为小档案了！快速又有效.

    [dmtsai@study ~]$ split [-bl] file PREFIX 
    选项与参数：
    -b ：后面可接欲分割成的档案大小，可加单位，例如b, k, m 等；
    -l ：以行数来进行分割。
    PREFIX ：代表前置字元的意思，可作为分割档案的前导文字。

    范例一：我的/etc/services有六百多K，若想要分成300K一个档案时？
    [dmtsai@study ~]$ cd /tmp; split -b 300k /etc/services services 
    [dmtsai@study tmp]$ ll -k services* 
    -rw-rw-r--. 1 dmtsai dmtsai 307200 Jul 9 22:52 services aa 
    -rw-rw-r--. 1 dmtsai dmtsai 307200 Jul 9 22:52 services ab 
    -rw-rw-r--. 1 dmtsai dmtsai 55893 Jul 9 22:52 services ac 
    #那个档名可以随意取的啦！我们只要写上前导文字，小档案就会以
    # xxxaa, xxxab, xxxac 等方式来建立小档案的！

    范例二：如何将上面的三个小档案合成一个档案，档名为servicesback 
    [dmtsai@study tmp]$ cat services* >> servicesback 
    #很简单吧？就用资料流重导向就好啦！简单！

# 19. head 和tail

    $ head -n 20 #显示前20行
    $ tail -n 10 #显示最后10行

# 20. echo 知多少

- echo -n 不换行输出

        $echo -n "123"
        $echo "456"

    最终输出   
    123456

    而不是  
    123  
    456

- echo -e 处理特殊字符

    若字符串中出现以下字符，则特别加以处理，而不会将它当成一般文字输出：

        \a 发出警告声；
        \b 删除前一个字符；
        \c 最后不加上换行符号；
        \f 换行但光标仍旧停留在原来的位置；
        \n 换行且光标移至行首；
        \r 光标移至行首，但不换行；
        \t 插入tab；
        \v 与\f相同；
        \\ 插入\字符；
        \nnn 插入nnn（八进制）所代表的ASCII字符；

    例子：

        $echo -e "a\bdddd"
        dddd

        $echo -e "a\adddd" //输出同时会发出报警声音
        adddd


        $echo -e "a\ndddd" //自动换行
        a
        dddd

# 21. test 测试
比如测试文件是否存在

    $ test -e version.txt
    $ echo $?
    # test会将结果存储到$?中，如果存在则输出0

|测试的标志|代表的意义|
|---|---|
1.『档案类型』判断，如test -e filename 表示存在否|-
-e  |该『档名』是否存在？(常用)
-f  |该『档名』是否存在且为档案(file)？(常用)
-d  |该『档名』是否存在且为目录(directory)？(常用)
-b  |该『档名』是否存在且为一个block device 装置？
-c  |该『档名』是否存在且为一个character device 装置？
-S  |该『档名』是否存在且为一个Socket 档案？
-p  |该『档名』是否存在且为一个FIFO (pipe) 档案？
-L  |该『档名』是否存在且为一个连结档？

2.关于档案的权限侦测，如test -r filename 表示可读否(但root 权限常有例外)|-
-r  |侦测该档名是否存在且具有『可读』的权限？
-w  |侦测该档名是否存在且具有『可写』的权限？
-x  |侦测该档名是否存在且具有『可执行』的权限？
-u  |侦测该档名是否存在且具有『SUID』的属性？
-g  |侦测该档名是否存在且具有『SGID』的属性？
-k  |侦测该档名是否存在且具有『Sticky bit』的属性？
-s  |侦测该档名是否存在且为『非空白档案』？

3. 两个档案之间的比较，如： test file1 -nt file2|-
-nt |(newer than)判断file1 是否比file2 新
-ot |(older than)判断file1 是否比file2 旧
-ef |判断file1 与file2 是否为同一档案，可用在判断hard link 的判定上。主要意义在判定，两个档案是否均指向同一个inode 哩！

4. 关于两个整数之间的判定，例如test n1 -eq n2|-
-eq |两数值相等(equal)
-ne |两数值不等(not equal)
-gt |n1 大于n2 (greater than)
-lt |n1 小于n2 (less than)
-ge |n1 大于等于n2 (greater than or equal)
-le |n1 小于等于n2 (less than or equal)

|5. 判定字串的资料 |-|
|--|--|
test -z string  |判定字串是否为0 ？若string 为空字串，则为true 
test -n string  |判定字串是否非为0 ？若string为空字串，则为false。 注： -n亦可省略
test str1 == str2   |判定str1 是否等于str2 ，若相等，则回传true
test str1 != str2   |判定str1 是否不等于str2 ，若相等，则回传false

6. 多重条件判定，例如： test -r filename -a -x filename|-
-a  |(and)两状况同时成立！例如test -r file -a -x file，则file 同时具有r 与 x 权限时，才回传true。
-o  |(or)两状况任何一个成立！例如test -r file -o -x file，则file 具有r 或 x 权限时，就可回传true。
!   |反相状态，如test ! -x file ，当file 不具有x 时，回传true

# End. 一些常识

- 命令行的 \ 就是换行作用，在要输入一条很长很长的命令时候使用。
- 上次的操作指令历史会存储到**~/.bash_history**文件里面。本次的操作要登出后才会存储。一个身份多个登录，前面的内容会被最后一个登出者覆盖。。
- [ctrl]+u : 从游标处向前删除指令串([ctrl]+u) (方向和vim的hjkl一样)
- [ctrl]+k : 向后删除指令串([ctrl]+k)
- [ctrl]+a : 让游标移动到整个指令串的最前面([ctrl]+a) 
- [ctrl]+e : 或最后面([ctrl]+e)
- echo $$ : 得到shell 的PID，因为$也是系统变量
- echo $? : 得到上个执行指令的回传值，正确输出0
- login shell配置读取顺序：/etc/profile （会引入/etc/profile.d/*.sh）， ~/.bash_profile > ~/.bash_login > ~/.profile（三个只会读取一个，按优先级从高到低读取）
- non-login shell仅读取 ~/.bashrc
- \>, >> : 资料流重导向：输出导向，分别是『取代』与『累加』
- 1> ：以覆盖的方法将『正确的资料』输出到指定的档案或装置上；（『 1>> 』以及『 2>> 』中间是没有空格的）
- 1>>：以累加的方法将『正确的资料』输出到指定的档案或装置上；
- 2> ：以覆盖的方法将『错误的资料』输出到指定的档案或装置上；
- 2>>：以累加的方法将『错误的资料』输出到指定的档案或装置上；
- nl : 显示文件内容，同时显示行号。 
- echo "scale=10;4*a(1)" | bc -lq #输出pi 

# Ref
> [鸟哥的Linux私房菜](http://linux.vbird.org/linux_basic/0320bash.php#bash)
> [Linux命令之trap - 在脚本中处理信号](http://codingstandards.iteye.com/blog/836588)
