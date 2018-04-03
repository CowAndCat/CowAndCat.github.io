---
layout: post
title: Shell 程序管理
category: linux
comments: false
---

# 1. 将『目前』的工作丢到背景中『暂停』：[ctrl]-z

想个情况：如果我正在使用vim ，却发现我有个档案不知道放在哪里，需要到bash 环境下进行搜寻，此时是否要结束vim 呢？呵呵！当然不需要啊！只要暂时将vim 给他丢到背景当中等待即可。例如以下的案例：

    [root@study ~]# vim ~/.bashrc 
    #在vim的一般模式下，按下[ctrl]-z这两个按键
    [1]+ Stopped vim ~/.bashrc

    [root@study ~]#    <==顺利取得了前景的操控权！
    [root@study ~]# find / -print 
    ....(输出省略)....

    # 此时萤幕会非常的忙碌！因为萤幕上会显示所有的档名。请按下[ctrl]-z 暂停
    [2]+ Stopped find / -print

在vim的一般模式下，按下[ctrl]及z这两个按键，萤幕上会出现[1] ，表示这是第一个工作，而那个+代表最近一个被丢进背景的工作，且目前在背景下预设会被取用的那个工作(与fg这个指令有关)！  
而那个Stopped则代表目前这个工作的状态。在预设的情况下，使用[ctrl]-z丢到背景当中的工作都是『暂停』的状态喔！

# 2. 观察目前的背景工作状态： jobs

    [root@study ~]# jobs [-lrs] 
    选项与参数：
    -l ：除了列出job number 与指令串之外，同时列出PID 的号码；
    -r ：仅列出正在背景run 的工作；
    -s ：仅列出正在背景当中暂停(stop) 的工作。

    范例一：观察目前的bash当中，所有的工作，与对应的PID 
    [root@study ~]# jobs -l
    [1]- 14566 Stopped vim ~/.bashrc
    [2]+ 14567 Stopped find / -print

+代表最近被放到背景的工作号码， -代表最近最后第二个被放置到背景中的工作号码。 而超过最后第三个以后的工作，就不会有+/-符号存在了！

上例表示，目前我有两个工作在背景当中，两个工作都是暂停的，而如果我仅输入fg时，那么那个[2]会被拿到前景当中来处理。

# 3. 将后台工作拿到前台来处理：fg

    [root@study ~]# fg %jobnumber 
    选项与参数：
    %jobnumber ：jobnumber 为工作号码(数字)。注意，那个% 是可有可无的！

另外，如果输入『fg -』则代表将-号的那个工作号码拿出来，上面就是[2]-那个工作号码啦！

# 4. 让工作在后台的状态变成运作中： bg
用法和fg一样，只不过是启动后台的程序。

# 5. 管理后台的工作： kill

    [root@study ~]# kill -signal %jobnumber 
    [root@study ~]# kill -l 
    选项与参数：
    -l ：这个是L 的小写，列出目前kill 能够使用的讯号(signal) 有哪些？
    signal ：代表给予后面接的那个工作什么样的指示啰！用man 7 signal 可知：
      -1 ：重新读取一次参数的设定档(类似reload)；
      -2 ：代表与由键盘输入[ctrl]-c 同样的动作；
      -9 ：立刻强制删除一个工作；
      -15：以正常的程序方式终止一项工作。与-9 是不一样的。

注意，-9这个signal通常是用在『强制删除一个不正常的工作』时所使用的，-15则是以正常步骤结束一项工作(15也是预设值)。

kill后面直接加数字与加上%number的情况是不同的：数字用于PID；%是专门用在工作控制的，仅在终端线程里生效。

**总结： 与job control 有关的按键与关键字有： &, [ctrl]-z, jobs, fg, bg, kill %n 等；**

# 6. 离线管理问题
如果你是以远端连线方式连接到你的Linux主机，并且将工作以&的方式放到后台去，请问，在工作尚未结束的情况下你离线了，该工作还会继续进行吗？  
答案是『否』！不会继续进行，而是会被中断掉。

解决：1. 用at，将工作放置到系统背景。 2. 用nohup，并搭配&来使用，语法如下：

    [root@study ~]# nohup [指令与参数]    <==在终端机前景中工作 
    [root@study ~]# nohup [指令与参数] &  <==在终端机背景中工作

# 7. 进程查看

## 7.1 ps ：将某个时间点的程序运作情况撷取下来

    [root@study ~]# ps aux  <==观察系统所有的程序资料 
    [root@study ~]# ps -lA  <==也是能够观察所有系统的资料 
    [root@study ~]# ps axjf  <==连同部分程序树状态
    选项与参数：
    -A ：所有的process 均显示出来，与-e 具有同样的效用；
    -a ：不与terminal 有关的所有process ；
    -u ：有效使用者(effective user) 相关的process ；
    x ：通常与a 这个参数一起使用，可列出较完整资讯。
    输出格式规划：
    l ：较长、较详细的将该PID 的的资讯列出；
    j ：工作的格式(jobs format)
    -f ：做一个更为完整的输出。

- 仅观察自己的bash 相关程序： ps -l
- 观察系统所有程序： ps aux

各项含义如下：

    F：代表这个程序旗标(process flags)，说明这个程序的总结权限，常见号码有：
        若为4 表示此程序的权限为root ；
        若为1则表示此子程序仅进行复制(fork)而没有实际执行(exec)。

    S：代表这个程序的状态(STAT)，主要的状态有：
    R (Running)：该程式正在运作中；
    S (Sleep)：该程式目前正在睡眠状态(idle)，但可以被唤醒(signal)。
    D ：不可被唤醒的睡眠状态，通常这支程式可能在等待I/O 的情况(ex>列印)
    T ：停止状态(stop)，可能是在工作控制(背景暂停)或除错(traced) 状态；
    Z (Zombie)：僵尸状态，程序已经终止但却无法被移除至记忆体外。

    UID/PID/PPID：代表『此程序被该UID 所拥有/程序的PID 号码/此程序的父程序PID 号码』

    C：代表CPU 使用率，单位为百分比；

    PRI/NI：Priority/Nice的缩写，代表此程序被CPU所执行的优先顺序，数值越小代表该程序越快被CPU执行。

    ADDR/SZ/WCHAN：都与记忆体有关，
        ADDR 是kernel function，指出该程序在记忆体的哪个部分，如果是个running 的程序，一般就会显示『 - 』  
        SZ 代表此程序用掉多少记忆体 
        WCHAN 表示目前程序是否运作中，同样的， 若为- 表示正在运作中。

    TTY：登入者的终端机位置，若为远端登入则使用动态终端介面(pts/n)；

    TIME：使用掉的CPU 时间，注意，是此程序实际花费CPU 运作的时间，而不是系统时间；

    CMD：就是command 的缩写，造成此程序的触发程式之指令为何。

命令也可以通过 ll /proc 来代替。


## 7.2 僵尸程序

通常，造成僵尸程序的成因是因为该程序应该已经执行完毕，或者是因故应该要终止了， 但是该程序的父程序却无法完整的将该程序结束掉，而造成那个程序一直存在记忆体当中。如果你发现在某个程序的CMD 后面还接上<defunct> 时，就代表该程序是僵尸程序啦，例如：

    apache 8683 0.0 0.9 83384 9992 ? Z 14:33 0:00 /usr/sbin/httpd <defunct>

当系统不稳定的时候就容易造成所谓的僵尸程序，可能是因为程式写的不好，或者是使用者的操作习惯不良等等所造成。

如果你发现系统中很多僵尸程序时，记得要找出该程序的父程序，然后好好的做个追踪，好好的进行主机的环境最佳化啊！看看有什么地方需要改善的，不要只是直接将他kill 掉而已呢！

事实上，通常僵尸程序都已经无法控管，而直接是交给systemd 这支程式来负责了，偏偏systemd 是系统第一支执行的程式，他是所有程式的父程式！我们无法杀掉该程式的(杀掉他，系统就死掉了！)，所以啰，如果产生僵尸程序，而系统过一阵子还没有办法透过核心非经常性的特殊处理来将该程序删除时，那你只好透过reboot 的方式来将该程序抹去了！

## 7.3 top: 动态观察程序的变化

相对于ps 是撷取一个时间点的程序状态， top 则可以持续侦测程序运作的状态.

    [root@study ~]# top [-d数字] | top [-bnp] 
    选项与参数：
    -d ：后面可以接秒数，就是整个程序画面更新的秒数。预设是5 秒；
    -b ：以批次的方式执行top ，还有更多的参数可以使用喔！
          通常会搭配资料流重导向来将批次的结果输出成为档案。
    -n ：与-b 搭配，意义是，需要进行几次top 的输出结果。
    -p ：指定某些个PID 来进行观察监测而已。
    在top 执行过程当中可以使用的按键指令：
        ? ：显示在top 当中可以输入的按键指令；
        P ：以CPU 的使用资源排序显示；
        M ：以Memory 的使用资源排序显示；
        N ：以PID 来排序喔！
        T ：由该Process 使用的CPU 时间累积(TIME+) 排序。
        k ：给予某个PID 一个讯号(signal)
        r ：给予某个PID 重新制订一个nice 值。
        q ：离开top 软体的按键。

top -p PID 能查看单一进程的动态使用信息。

## 7.4 Priority 与Nice 值

PRI值越低代表越优先的意思。不过这个PRI值是由核心动态调整的，使用者无法直接调整PRI值的。想要调整程序的优先执行序时，就得要透过Nice 值.

PRI与NI的相关性如下：

    PRI(new) = PRI(old) + nice

- nice 值可调整的范围为-20 ~ 19 ；
- root 可随意调整自己或他人程序的Nice 值，且范围为-20 ~ 19 ；
- 一般使用者仅可调整自己程序的Nice 值，且范围仅为0 ~ 19 (避免一般用户抢占系统资源)；
- 一般使用者仅可将nice 值越调越高，例如本来nice 为5 ，则未来仅能调整到大于5；

有两种方式调整nice值：
- 一开始执行程式就立即给予一个特定的nice 值：用nice 指令；
- 调整某个已经存在的PID 的nice 值：用renice 指令。

    [root@study ~]# nice [-n数字] command 
    选项与参数：
    -n ：后面接一个数值，数值的范围-20 ~ 19。

    范例一：用root给一个nice值为-5 ，用于执行vim ，并观察该程序！
    [root@study ~]# nice -n -5 vim &
    [1] 19865

    [root@study ~]# renice [number] PID 
    选项与参数：
    PID ：某个程序的ID！不用输-n了。

## 7.5 free ：观察记忆体使用情况

    [root@study ~]# free [-b|-k|-m|-g|-h] [-t] [-s N -c N] 
    选项与参数：
    -b ：直接输入free 时，显示的单位是Kbytes，我们可以使用b(bytes), m(Mbytes)
          k(Kbytes), 及g(Gbytes) 来显示单位喔！也可以直接让系统自己指定单位(-h)
    -t ：在输出的最终结果，显示实体记忆体与swap 的总量。
    -s ：可以让系统每几秒钟输出一次，不间断的一直输出的意思！对于系统观察挺有效！
    -c ：与-s 同时处理～让free 列出几次的意思～

    范例一：显示目前系统的记忆体容量 
    [root@study ~]# free -m
    total used free shared buff/cache available
    Mem: 2848 346 1794 8 706 2263
    Swap: 1023 0 1023

Mem 那一行显示的是实体记忆体的量， Swap 则是记忆体置换空间的量。total 是总量， used 是已被使用的量， free 则是剩余可用的量。后面的shared/buffers/cached 则是在已被使用的量当中，用来作为缓冲及快取的量，这些shared/buffers/cached 的用量中，在系统比较忙碌时， 可以被释出而继续利用！因此后面就有一个available (可用的) 数值！

## 7.6 uname：查阅系统与核心相关资讯
uname -a : 输出系统的基本资讯 

## 7.7 uptime：观察系统启动时间与工作负载
显示出目前系统已经开机多久的时间,以及1, 5, 15分钟的平均负载.

这个uptime可以显示出top画面的最上面一行.

## 7.8 netstat ：追踪网路或插槽档

    [root@study ~]# netstat -[atunlp] 
    选项与参数：
    -a ：将目前系统上所有的连线、监听、Socket 资料都列出来
    -t ：列出tcp 网路封包的资料
    -u ：列出udp 网路封包的资料
    -n ：不以程序的服务名称，以埠号(port number) 来显示；
    -l ：列出目前正在网路监听(listen) 的服务；
    -p ：列出该网路服务的程序PID 

netstat : 列出目前系统已经建立的网路连线与unix socket状态.

netstat -tulnp ：找出目前系统上已在监听的网路连线及其PID

各列的含义：

    Proto ：网路的封包协定，主要分为TCP与UDP封包，相关资料请参考伺服器篇；
    Recv-Q：非由使用者程式连结到此socket 的复制的总bytes 数；
    Send-Q：非由远端主机传送过来的acknowledged 总bytes 数；
    Local Address ：本地端的IP:port 情况
    Foreign Address：远端主机的IP:port 情况
    State ：连线状态，主要有建立(ESTABLISED)及监听(LISTEN)；

## 7.9 vmstat ：侦测系统资源变化
vmstat 可以侦测『 CPU / 记忆体/ 磁碟输入输出状态』

    [root@study ~]# vmstat [-a] [延迟[总计侦测次数]]  <==CPU/记忆体等资讯 
    [root@study ~]# vmstat [-fs]                       <==记忆体相关 
    [root @study ~]# vmstat [-S单位]                   <==设定显示数据的单位 
    [root@study ~]# vmstat [-d]                        <==与磁碟有关 
    [root@study ~]# vmstat [-p分割槽]                 <==与磁碟有关
    选项与参数：
    -a ：使用inactive/active(活跃与否) 取代buffer/cache 的记忆体输出资讯；
    -f ：开机到目前为止，系统复制(fork) 的程序数；
    -s ：将一些事件(开机至目前为止) 导致的记忆体变化情况列表说明；
    -S ：后面可以接单位，让显示的资料有单位。例如K/M 取代bytes 的容量；
    -d ：列出磁碟的读写总量统计表
    -p ：后面列出分割槽，可显示该分割槽的读写总量统计表

vmstat 1 3 :统计目前主机CPU状态，每秒一次，共计三次！  
vmstat -d : 系统上面所有的磁碟的读写状态 

## 7.10 fuser：藉由档案(或档案系统)找出正在使用该档案的程序
能知道我的程序到底在这次启动过程中开启了多少档案

    [root@study ~]# fuser [-umv] [-k [i] [-signal]] file/dir 
    选项与参数：
    -u ：除了程序的PID 之外，同时列出该程序的拥有者；
    -m ：后面接的那个档名会主动的上提到该档案系统的最顶层，对umount 不成功很有效！
    -v ：可以列出每个档案与程序还有指令的完整相关性！
    -k ：找出使用该档案/目录的PID ，并试图以SIGKILL 这个讯号给予该PID；
    -i ：必须与-k 配合，在删除PID 之前会先询问使用者意愿！
    -signal：例如-1 -15 等等，若不加的话，预设是SIGKILL (-9) 啰！

fuser -uv /proc ： 找到所有使用到/proc这个档案系统的程序

## 7.11 lsof ：列出被程序所开启的档案档名
相对于fuser 是由档案或者装置去找出使用该档案或装置的程序，反过来说，lsof能查出某个程序开启或者使用的档案与装置。

    [root@study ~]# lsof [-aUu] [+d] 
    选项与参数：
    -a ：多项资料需要『同时成立』才显示出结果时！
    -U ：仅列出Unix like 系统的socket 档案类型；
    -u ：后面接username，列出该使用者相关程序所开启的档案；
    +d ：后面接目录，亦即找出某个目录底下已经被开启的档案！


- lsof +d /dev：列出目前系统上面所有的被启动的周边装置
- lsof -u root | grep bash： 列出属于root的bash这支程式所开启的档案 

## 7.12 pidof ：找出某支正在执行的程式的PID

pidof systemd rsyslogd ：列出目前系统上面systemd以及rsyslogd这两个程式的PID 

# 参考
>[鸟哥的Linux 私房菜-第十六章、程序管理与SELinux 初探](http://linux.vbird.org/linux_basic/0440processcontrol.php)