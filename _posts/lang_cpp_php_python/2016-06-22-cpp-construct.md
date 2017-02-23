---
layout: post
title: C++ 继承关系中的构造函数、析构函数的调用顺序
category: c_cpp
comments: false
---

# C++继承中构造函数的调用顺序

C++ 派生类的构造函数的调用顺序为：基类、对象成员类和派生类的构造函数

例子：

```
	class A{
	public: 
		A() {cout<<"A";}
	};

	class B{
	public: 
		B() {cout<<"B";}
	};

	class C:public A{
		B b;
	public: 
		C() {cout<<"C";}
	};

	int main(){
		C obj;
		return 0;
	}
	//输出是 ABC
	//C++ 派生类的构造函数的调用顺序为：基类、对象成员类和派生类的构造函数
```

# C++继承中析构函数的调用顺序

C++ 派生类的析构函数的调用顺序为：派生类、对象成员类和基类的析构函数

例子：

```
	class Base{
	private:
		char c;

	public:
		Base(char n):c(n){ }
		virtual ~Base(){cout<<c;}
	};

	class Der : public Base{
	private:
		char c;

	public:
		Der(char n): Base(n+1), c(n){ }
		~Der(){cout<<c;}
	};

	int main(){
		Der('X');
		return 0;
	}
	//输出：XY 
	//C++ 派生类的析构函数的调用顺序为：派生类、对象成员类和基类的析构函数
```

# C++多继承 对象初始化顺序

1. 首先，任何虚拟基类的构造函数按照它们被继承的顺序构造；
1. 其次，任何非虚拟基类的构造函数按照它们被继承的顺序构造；
1. 最后，任何成员对象的构造函数按照它们声明的顺序调用；

例子1：

```
	#include <iostream>
	using namespace std;
	class OBJ1{
	public:
		OBJ1(){ cout<<"OBJ1\n"; }
	};

	class OBJ2{
	public:
		OBJ2(){ cout<<"OBJ2\n";}
	}

	class Base1{
	public:
		Base1(){ cout<<"Base1\n";}
	}

	class Base2{
	public:
		Base2(){ cout <<"Base2\n"; }
	};

	class Base3{
	public:
		Base3(){ cout <<"Base3\n"; }
	};

	class Base4{
	public:
		Base4(){ cout <<"Base4\n"; }
	};

	class Derived :public Base1, virtual public Base2,public Base3, virtual public Base4{//继承顺序
	public:
		Derived() :Base4(), Base3(), Base2(),Base1(), obj2(), obj1(){//初始化列表
			cout <<"Derived ok.\n";
		}
	protected:
		OBJ1 obj1;//声明顺序
		OBJ2 obj2;
	};

	int main()
	{
		Derived aa;//初始化
		cout <<"This is ok.\n";
		return 0;
	}
```

结果：

```
Base2 //虚拟基类按照被继承顺序初始化
Base4 //虚拟基类按照被继承的顺序 
Base1 //非虚拟基类按照被继承的顺序初始化
Base3 //非虚拟基类按照被继承的顺序 
OBJ1 //成员函数按照声明的顺序初始化
OBJ2 //成员函数按照声明的顺序 
Derived ok. 
This is ok.
```

例子2：

```
	class B1   
	{
	public:  
		B1(int i){cout<<"consB1"<<i<<endl;}  
	};//定义基类B1

	class B2    
	{
	public:  
		B2(int j){cout<<"consB2"<<j<<endl;}  
	};//定义基类B2  

	class B3   
	{  
	public:  
		B3(){cout<<"consB3 *"<<endl;}  
	};//定义基类B3  

	class C: public B2, public B1, public B3   
	{
	public:
		C(int a,int b,int c,int d,int e):B1(a),memberB2(d),memberB1(c),B2(b)  
		{m=e; cout<<"consC"<<endl;}  
	private:  
		B1 memberB1;  
		B2 memberB2;  
		B3 memberB3;  
		int m;  
	};//继承类C  

	void main()  
	{ C  obj(1,2,3,4,5);  }//主函数  
```

运行结果：

```
	consB2 2 
	consB1 1
	consB3 *
	consB1 3
	consB2 4
	consB3 *
	consC
```
```
//先按照继承顺序：B2，B1，B3
//第一步：先继承B2,在初始化列表里找到B2(b)，打印"constB22"
//第二步：再继承B1,在初始化列表里找到B1(a)，打印"constB11"
//第三步：又继承B3,在初始化列表里找不到B3(x), 则调用B3里的默认构造函数B3()，打印"constB3 *" 

//再按照数据成员定义顺序：memberB1, memberB2, memberB3
//第四步：在初始化列表里找到memberB1(c),初始化一个B1对象，用c为参数，则调用B1的构造函数，打印"constB13" 
//第五步：在初始化列表里找到memberB2(d),初始化一个B2对象，用d为参数，则调用B2的构造函数，打印"constB24" 
//第六步：在初始化列表里找不到memberB3(x),则调用B3里的默认构造函数B3()，打印"constB3 *" 
//最后完成本对象初始化的剩下部分,也就是C自己的构造函数的函数体：{m=e; cout<<"consC"<<endl;} 
//第七步：打印"consC"
```

**为什么会有两次B3\*出现？**  

第一次是由于继承了B3，虽然在C的构造函数的初始化列表里你没看到B3(x)或者B3()，但并不代表B3的构造函数没有在发挥作用。

事实上，B3被隐性初始化了，因为B3的构造函数没有参数，所以写不写B3()都无所谓，这里恰好省略了。

B1，B2则都是显性初始化，因为它们都需要参数。第二次是因为C有数据成员memberB3，又一次，你没有在C的构造函数的初始化列表里看到你希望出现的memberB3()，很显然，这又是一次隐性初始化。B3的构造函数再次被暗中调用。每一次B3的构造函数被调用，都会打印出“consB3 \*”。两次被调用，自然打印两次“consB3 \*”。

# 在C++d继承中, 虚函数、纯虚函数与普通函数三者的区别

指向基类的指针可以指向派生类对象，当基类指针指向派生类对象时，这种指针只能访问派生对象从基类继承而来的那些成员，不能访问子类特有的元素，除非应用强类型转换.

例如有基类Ｂ和从Ｂ派生的子类Ｄ，则B*p;D  dd; p=&dd;是可以的，
指针p只能访问从基类派生而来的成员，不能访问派生类Ｄ特有的成员．因为基类不知道派生类中的这些成员.

例子：

```
	#include <iostream>
	using namespace std;

	class A
	{
	public:
	    virtual void out1()=0;  ///由子类实现
	    virtual ~A(){};
	    virtual void out2() ///默认实现
	    {
	        cout<<"A(out2)"<<endl;
	    }
	    void out3() ///强制实现
	    {
	        cout<<"A(out3)"<<endl;
	    }
	};

	class B:public A
	{
	public:
	    virtual ~B(){};
	    void out1()
	    {
	        cout<<"B(out1)"<<endl;
	    }
	    void out2()
	    {
	        cout<<"B(out2)"<<endl;
	    }
	    void out3()
	    {
	        cout<<"B(out3)"<<endl;
	    }
	};

	int main()
	{
	    A *ab=new B;
	    ab->out1();
	    ab->out2();
	    ab->out3();
	    cout<<"************************"<<endl;
	    B *bb=new B;
	    bb->out1();
	    bb->out2();
	    bb->out3();

	    delete ab;
	    delete bb;
	    return 0;
	}

```

结果：

```
	B(out1)
	B(out2)
	A(out3)
	**************************
	B(out1)
	B(out2)
	B(out3)
```