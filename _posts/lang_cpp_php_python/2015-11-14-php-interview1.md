---
layout: post
title: php面试题1
category: php
comments: false
---
##1、基础面试题
###1.1 如何设置form表单中的只读属性？
- 利用readonly 设置指定内容的只读属性
- 利用disabled 实现

###1.2 在什么情况下，$name 与 $_POST['name']可以通用？
在php.ini中register_globals=On时，它们才都可以获取form表单中表单元素name的值。

但是不建议开启这个全局变量，因为会带来安全隐患或其他问题。

###1.3 CSS的理解
CSS是W3C协议提出的层叠样式表（Cascading Style Sheet），为了弥补HTML在显示属性设定上的不足而制定的。

最大的用途是：实现内容和表现形式的分离，改变网页的整体表现形式，更容易进行外观维护，使HTML文档代码更为简练，缩短浏览器的加载时间。

###1.4 插入CSS的方式
- 在标签内部定义style属性
- 在head区域，定义一对`<style></style>`标记，在标记内部利用标签名称、类选择符、id选择符设置属性。
- 创建.css文件定义样式，之后再HTML页面中利用`<link>`标签引入文件。如：`<link type="text/css" rel="stylesheet" href="PATH">`

###1.5 在IE6下解决双倍边距的问题
IE6下的bug，margin-left:10px会解析成20px。

加上`display:inline;`.

###1.6 解决超链接被点击后hover样式不出现的问题？
只要对超链接样式属性进行正确的排序即可。顺序如下：
	
	link-visited-hover-action

代码如下：

	a:link{color:red; text-decoration:none}
	a:visited{...}
	a:hover{...}
	a:action{...}

###1.7 定义1px左右高度的容器
通过CSS样式定义的1px高度的区块往往是不能正常显示的，解决的方法如下：

在div标签中控制文字的行高，超出行高的内容设置为不显示：
	
	div{overflow:hidden|zoom:0.08 | line-height:1px;border:1px solid black}

###1.8 `<span>`和`<div>`的区别
- `<span>`是在HTML4.0时才加入，后者在3.0就有了。
- 同样作用于网页布局中，span是属于内联的，一般用于小模块的样式内联到HTML中；div是块级元素，用于组合大块的代码。

###1.9 编写代码，当鼠标划过文本框，自动选择文本框中的内容
`<input id="text" type="text" value="text">`

使用select方法：

`<input id="text" type="text" value="text" onmouseover="this.select()">`

扩展：当鼠标划过，自动情况内容：

`<input id="text" type="text" value="text" onclick="this.value=''">`

###1.10 php中文乱码解决
只要保证php.in、页面编码、文件编码（这个容易忽略）三者编码保持一致。
用utf-8时，文件需要保持为utf-8，并消息BOM格式，页面上设置：

	header("Content-Type: text/html; charset=utf-8");

或
	
	<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

utf-8与utf-8(无BOM)的区别 ：简单来说，BOM是用来表示文件字节是大端编码还是小端编码的三个字节，无BOM保存就是在文件开头没有这三个字节。

>[utf-8与utf-8(无BOM)的区别](http://afericazebra.blog.163.com/blog/static/30050408201211199298711/)

>BOM——Byte Order Mark，就是字节序标记。在UCS 编码中有一个叫做"ZERO WIDTH NO-BREAK SPACE"的字符，它的编码是FEFF。而FFFE在UCS中是不存在的字符，所以不应该出现在实际传输中。UCS规范建议我们在传输字节流前，先传输 字符"ZERO WIDTH NO-BREAK SPACE"。这样如果接收者收到FEFF，就表明这个字节流是Big-Endian的；如果收到FFFE，就表明这个字节流是Little- Endian的。因此字符"ZERO WIDTH NO-BREAK SPACE"又被称作BOM。

>UTF-8不需要BOM来表明字节顺序，但可以用BOM来表明编码方式。字符"ZERO WIDTH NO-BREAK SPACE"的UTF-8编码是EF BB BF。所以如果接收者收到以EF BB BF开头的字节流，就知道这是UTF-8编码了。

>UTF- 8编码的文件中，BOM占三个字节。如果用记事本把一个文本文件另存为UTF-8编码方式的话，用UE打开这个文件，切换到十六进制编辑状态就可以看到开 头的FFFE了。这是个标识UTF-8编码文件的好办法，软件通过BOM来识别这个文件是否是UTF-8编码，很多软件还要求读入的文件必须带BOM。

PHP不支持BOM。PHP在设计时就没有考虑BOM的问题，也就是说他不会忽略UTF-8编码的文件开头BOM的那三个字符。

##2、PHP基础面试题
###2.1 PHP的含义
是PHP Hypertext Preprocessor（PHP超文本预处理器）的缩写，是一种服务器端的、开源的、跨平台的、HTML嵌入式的脚步语言，语法混合了C、Java和Perl的特点，适合Web开发。

###2.2 PHP的优劣势
- 结构化（html嵌入式），把前端代码从程序分离出来、用面向对象编程特性编程
- 规范化。方法和变量命名，提高代码可读性。
- 安全性：php搞笑、灵活、没有固定的框架，但是它把安全问题留给了程序员。程序员要自行解决如跨脚步攻击（XSS）（可以用`htmlspecialchars`函数填补)、跨站请求伪造（CSRF）、代码注入漏洞、字符编码循环漏洞等。

它强大到足以成为在网络上最大的博客系统的核心（WordPress）！
它深邃到足以运行最大的社交网络（facebook）！
而它的易用程度足以成为初学者的首选服务器端语言！

###2.3 类型转换
用settype函数：
	`settype($var, 'bool');`

gettype获取类型

	gettype('1');返回的是string 
	gettype(1);返回的是integer 	

类型判断：is_string()、is_object等。

‘===’比较运算符，表示数值上相等，而且两者的类型也一样

###2.4 PHP基础
- 在 PHP 中，所有用户定义的函数、类和关键词（例如 if、else、echo 等等）都对大小写**不敏感**；（ECHO  和echo是一样的）所有变量都对大小写敏感。（ $color、$COLOR 以及 $coLOR 被视作三个不同的变量）
- Local 和 Global 作用域：
函数之外声明的变量拥有 Global 作用域，**只能**在函数以外进行访问。  
函数内部声明的变量拥有 LOCAL 作用域，只能在函数内部进行访问。  
下列的输出，只有10。

		<?php
		$x=5; // global scope
		  
		function myTest() {
		   $y=10; // local scope
		   echo "<p>在函数内部测试变量：</p>";
		   echo "变量 x 是：$x";
		   echo "<br>";
		   echo "变量 y 是：$y";
		} 
- global 关键词用于访问函数内的全局变量。
要做到这一点，请在（函数内部）变量前面使用 global 关键词：

		function myTest() {
		  global $x,$y;
		  $y=$x+$y;
		}
- PHP 同时在名为 $GLOBALS[index] 的数组中存储了所有的全局变量。下标存有变量名。这个数组在函数内也可以访问，并能够用于直接更新全局变量。

		function myTest() {
		  $GLOBALS['y']=$GLOBALS['x']+$GLOBALS['y'];
		} 
- 在 PHP 中，有两种基本的输出方法：echo 和 print。
  echo 和 print 之间的差异：
	- echo - 能够输出一个以上的字符串,如：`echo "This", " string", " was", " made", " with multiple parameters.";`
	- print - 只能输出一个字符串，并始终返回 1
	- 提示：echo 比 print 稍快，因为它不返回任何值。
	
	```
	$cars=array("Volvo","BMW","SAAB");  ```
	```echo "My car is a {$cars[0]}";	
	```
-  var_dump() 会返回变量的数据类型和值

		$x = 5985;
		var_dump($x);
		//输出：int(5985) 

- 如需设置常量，请使用 define() 函数 - 它使用三个参数：
	- 首个参数定义常量的名称
	- 第二个参数定义常量的值
	- 可选的第三个参数规定常量名是否对大小写敏感。默认是 false。 
	 
	下例创建了一个对大小写敏感的常量，值为 "Welcome to W3School.com.cn!"：

		<?php
		define("GREETING", "Welcome to W3School.com.cn!");
		echo GREETING;
		//define("GREETING", "Welcome to W3School.com.cn!", true);//不敏感
		?>
- 字符串连用`.`,如:`$a = "Hello";$b = $a . " world!";echo $b; // 输出 Hello world!`

- !==运算符：不全等（完全不同）	$x !== $y	如果 $x 不等于 $y，且它们类型不相同，则返回 true。

- 定义函数：

		function functionName() {
		  被执行的代码;
		}

- 在 PHP 中， array() 函数用于创建数组。
	
	- 索引数组 - 带有数字索引的数组：`$cars=array("Volvo","BMW","SAAB");//索引是自动分配的（索引从 0 开始`
	- 关联数组 - 带有指定键的数组:`$age=array("Peter"=>"35","Ben"=>"37","Joe"=>"43");`
	- 多维数组 - 包含一个或多个数组的数组:

			$cars = array
			  (
			  array("Volvo",22,18),
			  array("BMW",15,13),
			  array("Saab",5,2),
			  array("Land Rover",17,15)
			  );

- PHP 中的许多预定义变量都是“超全局的”，这意味着它们在一个脚本的全部作用域中都可用。在函数或方法中无需执行 global $variable; 就可以访问它们。
	- $GLOBALS:引用全局作用域中可用的全部变量
	- $_SERVER:保存关于报头、路径和脚本位置的信息,如：`$_SERVER['PHP_SELF']	返回当前执行脚本的文件名。`
	- **$_REQUEST**:用于收集 HTML 表单提交的数据.
	- ** $_POST**: 广泛用于收集提交 method="post" 的 HTML 表单后的表单数据。$_POST 也常用于传递变量。
	- **$_GET**
	- $_FILES
	- $_ENV
	- $_COOKIE
	- $_SESSION
-  Date() 函数把时间戳格式化为更易读的日期和时间。`date(format,timestamp)`  
mktime() 函数返回日期的 Unix 时间戳:`mktime(hour,minute,second,month,day,year)`  

		$d=mktime(9, 12, 31, 6, 10, 2015);
		echo "创建日期是 " . date("Y-m-d h:i:sa", $d);

	strtotime() 函数用于把人类可读的字符串转换为 Unix 时间。
- include vs. require:require 语句同样用于向 PHP 代码中引用文件。
不过，include 与 require 有一个巨大的差异：如果用 include 语句引用某个文件并且 PHP 无法找到它，脚本会继续执行.


###2.5 php表单、cookie
	
	<html>
	<body>
	
	<form action="welcome.php" method="post">
	Name: <input type="text" name="name"><br>
	E-mail: <input type="text" name="email"><br>
	<input type="submit">
	</form>
	
	</body>
	</html>

welcome.php：

	<html>
	<body>
	
	Welcome <?php echo $_POST["name"]; ?><br>
	Your email address is: <?php echo $_POST["email"]; ?>
	
	</body>
	</html>

输出：
	
	Welcome John
	Your email address is john.doe@example.com

- 防止XSS，用test_input函数来检查传递的数据
	 
		function test_input($data) {
		  $data = trim($data);//去除用户输入数据中不必要的字符（多余的空格、制表符、换行）
		  $data = stripslashes($data);//删除用户输入数据中的反斜杠（\）
		  $data = htmlspecialchars($data);//把特殊字符转换为 HTML 实体
		  return $data;
		}

- 如何创建 cookie？  
setcookie() 函数用于设置 cookie。`setcookie(name, value, expire, path, domain);`
注释：setcookie() 函数必须位于 <html> 标签之前。

	cookie 常用于识别用户。cookie 是服务器留在用户计算机中的小文件。每当相同的计算机通过浏览器请求页面时，它同时会发送 cookie。通过 PHP，您能够创建并取回 cookie 的值。

- 如果您的应用程序涉及不支持 cookie 的浏览器，您就不得不采取其他方法在应用程序中从一张页面向另一张页面传递信息。一种方式是从表单传递数据（有关表单和用户输入的内容，稍早前我们已经在本教程中介绍过了）。



##3、PHP中利用JQuery的Ajax实现功能

Asynchronous JavaScript And XML（异步 JavaScript 及 XML）

通过下面的这个例子进行学习吧。

在文本框中输入一个年份，判断其生肖，并在文本框输出。

	<html>
	<head>
	<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
	</head>
	<body>
	<input type="text" value="年份格式1990" onclick="this.select()">
	<input type="submit" value="Judge">
	<span></span>
	
	<script type="text/javascript" src="jquery-2.1.4.min.js"></script>
	<script>
	$(function(){
		$("input:last").click(function(){
			$.get("in.php", {
				number:$("input:first").val()
			}, function(data){
				$("span").text(data);
				
			});
		});
	});
	</script>
	
	</body>
	</html>

服务器端in.php文件

	<?php
	if(isset($_GET['number'])){
		$array=array("猪","鼠","牛","虎","兔","龙","蛇","马","羊","猴","鸡","狗");
		foreach($array as $key=>$value){
			if(ceil($_GET['number']%12)==$key){
				echo $value;
			}
		}
	}
	?>

##4、PHP基础
###4.1 程序的运行结果

	<?php
	$strA=null;
	$strB=false;
	echo $strA==$strB ? '相等':'不相等';

	$strC='';
	$strD=0;
	echo $strC==$strD ? '相等':'不相等';
	?>

结果都是“相等”，null和false都属于空值，''和0依然是空。

###4.2 大小写
PHP在定义变量的时候、获取变量的时候是不分大小写的，获取form表单中的值时是区分大小写的。

###4.3 单双引号
都是定义字符串的方式。

单引号包含的变量按普通字符串输出；

双引号所包含的变量会自动被替换成实际数值。
###4.4 操作符优先级
'.'优先级高于'+',如下面的程序

	<?php echo 'Test'.1+2.'45';?>

输出是'245',('Test'.1)+(2.'45')，由于前面不是数字，运算结果就是0，后面是245

###4.5 include和require
还有include_once和require_once函数，用来在当前代码中添加库代码。

两者的区别：include执行到该语句的时候再包含，如果出现错误，只会给出警告；后者出现错误的时候抛出致命错误，并终止程序执行。

`include 'a.php';`

当同一页面包含两个相同的自定义函数时，会导致运行出错。

###4.6 PHP的弱类型特点
	<?php
	$a="hello";
	if($a==0){
		echo "hello";
	}else{
	}
	?>

会输出hello！$a在比较时隐性地转换成了数字型变量0.

php不会严格检验传入的变量类型，也可以将变量自由的转换类型。

###4.7 PHP 重定向
是header函数，不是redirect()!!

重定向方法总结：  
方法一：`<?php header("Location: viewNote.php"); ?> `  
方法二：`echo "<scrīpt>window.location ="$PHP_SELF";</script>";`       
方法三：`echo "<META HTTP-EQUIV="Refresh" CONTENT="0; URL=index.php">";`

header()函数必须放在php程序的开头部分，而且之前不能有另外的 header() 函数或者 setcookie() 被调用，如果是带有网页输出，本语句必须放在 <HEAD></HEAD>标记之前。

###4.8 `mysql_fetch_row()`和`mysql_fetch_array()`
前者返回的是数字索引数组，只能通过数字索引获取元素值；

后者返回关联数组，既能通过数字，又能通过字符串索引来获取元素值。


###4.9 php中的静态变量和全局变量
全局变量：定义在所有函数以外的变量，函数内部通过两种方式访问全局变量：

	global $gvar;
	globals['gvar'];

静态变量：能够在函数调用之后仍保留变量值，当再次回到其作用域时，又可以继续使用原来的值。

###4.10 echo(),print()和print_r()区别
print：只能打印简单类型变量值

print_r:打印复杂类型变量的值（如数组、对象）

echo：输出一个或多个字符串

###4.11 中文字符串截取(mb_substr)
`mb_substr( $str, $start, $length, $encoding ) `

	<?php 
	$str='脚本之家：http://www.jb51.net'; 
	echo mb_substr($str,0,4,'utf-8');//截取头5个字，假定此代码所在php文件的编码为utf-8 
	?> 
结果显示：脚本之家 

获取中文长度:mb_strlen() :

`mb_strlen( $str, $encoding ) `

###4.12 其他字符串操作
- explode()	把字符串打散为数组。
- implode()	返回由数组元素组合成的字符串。别名：join()
- htmlspecialchars_decode()	把一些预定义的 HTML 实体转换为字符。
- htmlspecialchars()	把一些预定义的字符转换为 HTML 实体
- sprintf()	把格式化的字符串写入变量中。
- printf()	输出格式化的字符串。
- trim()	移除字符串两侧的空白字符和其他字符。
 
###4.13 php 5的构造函数和析构函数
__construct & __destruct

`parent::`调用父类的方法，如`parent::__construct()`

`self`——用于在类内引用静态的成员属性和方法。

###4.14 错误与异常
`error_reporting(2047);`//输出所有错误和警告，2047即E_ALL  
`display_errors` 默认是NO，控制脚本执行期间出现的错误信息是否显示给用户。

###4.15 fopen()和@fopen()的区别
@用于屏蔽程序中可能或存在的一些错误信息，可以防止用户看到错误报告的具体内容而产生恶意想法。

##5、PHP高级
###5.1 什么是 PHP 过滤器？
PHP 过滤器用于验证和过滤来自非安全来源的数据。  
验证和过滤用户输入或自定义数据是任何 Web 应用程序的重要组成部分。
设计 PHP 的过滤器扩展的目的是使数据过滤更轻松快捷。

什么是外部数据？1.来自表单的输入数据,2.Cookies,3.服务器变量,4.数据库查询结果.

**函数和过滤器**  
如需过滤变量，请使用下面的过滤器函数之一：

- `filter_var()` - 通过一个指定的过滤器来过滤单一的变量
- `filter_var_array()` - 通过相同的或不同的过滤器来过滤多个变量
- `filter_input` - 获取一个输入变量，并对它进行过滤
- `filter_input_array` - 获取多个输入变量，并通过相同的或不同的过滤器对它们进行过滤
在下面的例子中，我们用 `filter_var()` 函数验证了一个整数：

		<?php
		$int = 123;
		
		if(!filter_var($int, FILTER_VALIDATE_INT))
		 {
		 echo("Integer is not valid");
		 }
		else
		 {
		 echo("Integer is valid");
		 }

###5.2 Mysql操作

	$con = mysql_connect("localhost","peter","abc123") or die('Could not connect: ' . mysql_error());
	
	mysql_query("CREATE DATABASE my_db",$con);// Create database
		
	mysql_select_db("my_db", $con);

	// Create table in my_db database
	$sql = "CREATE TABLE Persons 
	(
	FirstName varchar(15),
	LastName varchar(15),
	Age int
	)";
	mysql_query($sql,$con);

	//select
	$result = mysql_query("SELECT * FROM Persons ORDER BY age");
	
	while($row = mysql_fetch_array($result))
	  {
	  echo $row['FirstName'];
	  echo " " . $row['LastName'];
	  echo " " . $row['Age'];
	  echo "<br />";
	  }

	//update
	mysql_query("UPDATE Persons SET Age = '36'
	WHERE FirstName = 'Peter' AND LastName = 'Griffin'");

	//delete
	mysql_query("DELETE FROM Persons WHERE LastName='Griffin'");

	mysql_close($con);

###5.3 二进制安全
二进制安全功能（binary-safe function）是指在一个二进制文件上所执行的不更改文件内容的功能或者操作。这能够保证文件不会因为某些操作而遭到损坏。

（就是指函数的参数包括二进制数据的时候，函数依然能正常操作。）



**只关心二进制化的字符串,不关心具体格式.**只会严格的按照二进制的数据存取.不会妄图已某种格式解析数据.
有时候以错误的格式打开一个文件,会导致文件发生不可逆的损坏.而二进制安全就是说会避免这种损坏.


例如strlen，在输入数据里有\0的时候，并不会在此停止。所以可以说是二进制安全的。  
PHP的strlen是怎么都不会停的，直接返回这个字符串内部数据结构的length值。strlen( "te\0st\0test" ) = 10

再比如函数strcoll，它是非二进制安全。而strcmp则是二进制安全的。

它们的区别是：

	$string1 = "Hello";
	$string2 = "Hello\x00Hello";
	echo strcoll($string1, $string2); // 返回0, 由于是非二进制安全，误判为相等
	echo strcmp($string1, $string2); // 返回负数

###5.4 闭包
例子：
闭包的语法很简单，需要注意的关键字就只有use，use意思是**连接闭包和外界变量**。

	$a = function() use($b) {
	 
	}
What：
作用：
>[PHP的闭包](http://www.cnblogs.com/yjf512/archive/2012/10/29/2744702.html)
	
	- 减少foreach的循环的代码
	- 减少函数的参数
	- 解除递归函数
	- 关于延迟绑定

###5.5 PHP调试方法
- 增加中间变量或跟踪变量：例如输出
- 用注释语句排除法进行调试：注释掉代码，错误没有了，那么大致可以定位错误的位置
- 通过调试器：设置断点
- 分段对程序做流程图解析。


##6、php.ini
###6.1 safe_mode选项
开启该选项，PHP将检查当前脚本的拥有者是否和被操作文件的拥有者相同，将影响文件操作类函数和程序执行类函数，如：pathinfo/basename/fopen/exec等。

###6.2 上传文件
`file_uploads` 设置为NO  
`upload_tmp_dir` 默认是系统默认tmp目录  
`upload_max_filesize` 上传文件的最大值  
`max_execution_time` 一个指令所能执行的最大时间  
`memory_limit` 一个指令分配的内存空间 单位是M

##8、php框架
- Zend：Web2.0风格，功能强大，松耦合
- ThinkPHP
国内开发、原名FCS，开源
- Seagull 简单易用，适合web、命令行和GUI应用的PHP框架
- CodeIgniter 是一个简单快速的PHP MVC框架。EllisLab 的工作人员发布了 CodeIgniter。许多企业尝试体验过所有 PHP MVC 框架之后，CodeIgniter 都成为赢家，主要是由于它为组织提供了足够的自由支持，允许开发人员更迅速地工作。

###8.1 Zend Framework
常用组件：

- `Zend_Controller`
此模块为应用程序提供全面的控制。它将请求转化为特定的行为并确保其执行。
- `Zend_Db`
此模块基于 PHP 数据对象 (PDO) 并提供一种通用方式来访问数据库。
- `Zend_Feed`
此模块使使用 RSS 和 Atom 提要变得简单。
- `Zend_Filter`
此模块提供字符串过滤函数，如 isEmail() 和 getAlpha()。
- `Zend_InputFilter`
对于 Zend_Filter，此模块是用来操作数组的，如表单输入。
- `Zend_HttpClient`
此模块使您能轻易地执行 HTTP 请求。
- `Zend_Json`
此模块使您能够轻易地将 PHP 对象转换成 JavaScript 对象符号，反之亦然。
- `Zend_Log`
此模块提供通用日志功能。
##7、php垃圾处理机制
###7.1引用计数基本知识  
>http://php.net/manual/zh/features.gc.refcounting-basics.php

每个php变量存在一个叫"**zval**"的变量容器中。一个zval变量容器，除了包含**变量的类型和值，还包括两个字节的额外信息**。

- 第一个是"is_ref"，是个bool值，用来标识这个变量是否是属于引用集合(reference set)。通过这个字节，php引擎才能把普通变量和引用变量区分开来，由于php允许用户通过使用&来使用自定义引用，zval变量容器中还有一个内部引用计数机制，来优化内存使用。
- 第二个额外字节是"refcount"，用以表示指向这个zval变量容器的变量(也称符号即symbol)个数。所有的符号存在一个符号表中，其中每个符号都有作用域(scope)，那些主脚本(比如：通过浏览器请求的的脚本)和每个函数或者方法也都有作用域。

当一个变量被赋常量值时，就会生成一个zval变量容器，如下例这样：

Example #1 生成一个新的zval容器

	<?php
	$a = "new string";
	?>
在上例中，新的变量a，是在当前作用域中生成的。并且生成了类型为 string 和值为new string的变量容器。在额外的两个字节信息中，`is_ref`被默认设置为 FALSE，因为没有任何自定义的引用生成。"refcount" 被设定为 1，因为这里只有一个变量使用这个变量容器. 注意到当"refcount"的值是1时，`is_ref`的值总是FALSE. 如果你已经安装了» Xdebug，你能通过调用函数 `xdebug_debug_zval()`显示"refcount"和`"is_ref"`的值。

Example #4 减少引用计数

	<?php
	$a = "new string";
	$c = $b = $a;
	xdebug_debug_zval( 'a' );
	unset( $b, $c );
	xdebug_debug_zval( 'a' );
	?>
以上例程会输出：

	a: (refcount=3, is_ref=0)='new string'
	a: (refcount=1, is_ref=0)='new string'
如果我们现在执行 unset($a);，包含类型和值的这个变量容器就会从内存中删除。

变量容器在”refcount“变成0时就被销毁. 当任何关联到某个变量容器的变量离开它的作用域(比如：函数执行结束)，或者对变量调用了函数 unset()时，”refcount“就会减1.

**复合类型**  
当考虑像 array和object这样的复合类型时，事情就稍微有点复杂. 与 标量(scalar)类型的值不同，array和 object类型的变量把它们的成员或属性存在自己的**符号表**中。这意味着下面的例子将生成三个zval变量容器。

Example #5 Creating a array zval

	<?php
	$a = array( 'meaning' => 'life', 'number' => 42 );
	xdebug_debug_zval( 'a' );
	?>
以上例程的输出类似于：

	a: (refcount=1, is_ref=0)=array (
	   'meaning' => (refcount=1, is_ref=0)='life',
	   'number' => (refcount=1, is_ref=0)=42
	)
###7.2 回收周期
>http://php.net/manual/zh/features.gc.collecting-cycles.php

首先，我们先要建立一些基本规则，如果一个引用计数增加，它将继续被使用，当然就不再在垃圾中。如果引用计数减少到零，所在变量容器将被清除(free)。就是说，仅仅在引用计数减少到非零值时，才会产生垃圾周期(garbage cycle)。

其次，在一个垃圾周期中，通过检查引用计数是否减1，并且检查哪些变量容器的引用次数是零，来发现哪部分是垃圾。

算法中都是模拟删除、模拟恢复、真的删除，都使用简单的遍历即可（最典型的深搜遍历）。复杂度为执行模拟操作的节点数正相关。

当垃圾回收机制打开时，每当根缓存区存满时，就会执行上面描述的循环查找算法。根缓存区有固定的大小，可存10,000个可能根。 

##8. PHP内核探索
###8.1 变量的创建

PHP总共有三个模块：内核、Zend引擎、以及扩展层。

PHP内核用来处理请求、文件流、错误处理等相关操作；
Zend引擎（ZE）用以将源文件转换成机器语言，然后在虚拟机上运行它；
扩展层是一组函数、类库和流，PHP使用它们来执行一些特定的操作。
比如，我们需要mysql扩展来连接MySQL数据库； 当ZE执行程序时可能会需要连接若干扩展，这时ZE将控制权交给扩展，等处理完特定任务后再返还；最后，ZE将程序运行结果返回给PHP内核，它再将结果传送给SAPI层，最终输出到浏览器上。