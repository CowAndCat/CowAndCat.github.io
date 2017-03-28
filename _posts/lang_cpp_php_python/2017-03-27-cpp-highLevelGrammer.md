---
layout: post
title: C++ 高级语法
category: c_cpp
comments: false
---

# 1.  C++/C 宏定义（define）中\# \#\# 的含义

define 中的"#" "##" 一般是用来拼接字符串的，但是实际使用过程中，有哪些细微的差别呢，我们通过几个例子来看看。

"\#"是字符串化的意思，出现在宏定义中的#是把跟在后面的参数转成一个字符串；

eg：
    
    #define  strcpy__(dst, src)      strcpy(dst, #src)

    strcpy__(buff,abc)  相当于 strcpy__(buff,“abc”)

"\#\#"是连接符号，把参数连接在一起

    #define FUN(arg)     my##arg
    则     FUN(ABC)
    等价于  myABC

再看一个具体的例子

    #include <iostream>
         
    using namespace std;
             
    #define  OUTPUT(A) cout<<#A<<":"<<(A)<<endl;
             
    int main()
    {
        int a=1,b=2;
             
        OUTPUT(a);
        OUTPUT(b);
        OUTPUT(a+b);
             
        return 1;
    }

输出：
    
    a:1
    b:2
    a+b:3