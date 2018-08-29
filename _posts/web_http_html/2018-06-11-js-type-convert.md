---
layout: post
title: js类型转换
category: javascript
comments: false
--- 

JavaScript中的取值类型非常灵活。 一些例子：

    “10+"objects"   //=＞"10 objects".数字10转换成字符串
    "7"*"4"         //=＞28:两个字符串均转换为数字
    var n=1-"x";    //=＞NaN:字符串"x"无法转换为数字
    n+"objects"     //=＞"NaN objects":NaN转换为字符串"NaN”

## 一、 类型转换

下表简要说明了在JavaScript中如何进行类型转换。

![1](/images/201804/js_type_convert.png "js type convert")

## 二、转换和相等性

由于JavaScript可以做灵活的类型转换，因此其“==”相等运算符也随相等的含义灵活多变。例如，如下这些比较结果均是true：

    null==undefined //这两值被认为相等
    "0"==0          //在比较之前字符串转换成数字
    0==false        //在比较之前布尔值转换成数字
    "0"==false      //在比较之前字符串和布尔值都转换成数字      

隐式转换，例子：

    “x+""   //等价于String(x)
    +x      //等价于Number(x).也可以写成x-0
    !!x     //等价于Boolean(x).注意是双叹号

## 三、常用类型转换

### 3.1 数字到某进制字符串

    var n=17;
    binary_string=n.toString(2);    //转换为"10001"
    octal_string="0"+n.toString(8); //转换为"021"
    hex_string="0x"+n.toString(16); //转换为"0x11”

转回去

    parseInt("11",2);   //=＞3   (1*2+1)
    parseInt("ff",16);  //=＞255 (15*16+15)
    parseInt("zz",36);  //=＞1295 (35*36+35)
    parseInt("077",8);  //=＞63  (7*8+7)
    parseInt("077",10); //=＞77  (7*10+7)


### 3.2 浮点精度控制

    var n=123456.789;
    n.toFixed(0);   //"123457"
    n.toFixed(2);   //"123456.79"
    n.toFixed(5);   //"123456.78900"
    n.toExponential(1); //"1.2e+5"
    n.toExponential(3); //"1.235e+5"
    n.toPrecision(4);   //"1.235e+5"
    n.toPrecision(7);   //"123456.8"
    n.toPrecision(10);  //"123456.7890”

### 3.3 解析

    parseInt("3 blind mice")    //=＞3
    parseFloat("3.14 meters")   //=＞3.14”
    parseInt("-12.34")  //=＞-12
    parseInt("0xFF")    //=＞255
    parseInt("0xff")    //=＞255
    parseInt("-0XFF")   //=＞-255
    parseFloat(".1")    //=＞0.1
    parseInt("0.1")     //=＞0
    parseInt(".1")      //=＞NaN:整数不能以"."开始
    parseFloat("$72.47");//=＞NaN:数字不能以"$"开始”


## 参考
>(美)David Flanagan. “JavaScript权威指南（原书第6版）”