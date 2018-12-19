---
layout: post
title: Java 动态代理机制
category: java
comments: false
---
Java中代理的实现一般分为三种：JDK静态代理、JDK动态代理以及CGLIB动态代理。在Spring的AOP实现中，主要应用了JDK动态代理以及CGLIB动态代理。

## 一、静态代理

代理一般实现的模式为JDK静态代理：
创建一个接口，然后创建被代理的类实现该接口并且实现该接口中的抽象方法。之后再创建一个代理类，同时使其也实现这个接口。在代理类中持有一个被代理对象的引用，而后在代理类方法中调用该对象的方法。
其实就是代理类为被代理类预处理消息、过滤消息并在此之后将消息转发给被代理类，之后还能进行消息的后置处理。代理类和被代理类通常会存在关联关系(即上面提到的持有的被代理对象的引用)，代理类本身不实现服务，而是通过调用被代理类中的方法来提供服务。

接口
    
    public interface HelloInterface {
        void sayHello();
    }

实现类：

    public class Hello implements HelloInterface {
        @Override
        public void sayHello() {
            System.out.println("Hello~");
        }
    }


代理类：

    public class HelloProxy implements HelloInterface {
        private HelloInterface hello = new Hello();

        @Override
        public void sayHello() {
            System.out.println("Proxy do before");
            hello.sayHello();
            System.out.println("Proxy do after");
        }
    }

调用：

    public class Client {
        public static void main(String[] args) {
            HelloInterface proxy = new HelloProxy();
            proxy.sayHello();
        }
    }

    /** 
    输出结果：
        Proxy do before
        Hello~
        Proxy do after
    */

上述例子可以看到，使用JDK静态代理很容易就完成了对一个类的代理操作。

上述的例子和装饰器模式很像，都会持有一个超类对象，区别在于装饰器需要从外部传入需要装饰的对象，且目的在于增加内容。

静态代理的缺点：由于代理只能为一个类服务，如果需要代理的类很多，那么就需要编写大量的代理类，比较繁琐。

## 二、动态代理

### 2.1 例子和分析
JDK动态代理其实也是基本接口实现的。因为通过接口指向实现类实例的多态方式，可以有效地将具体实现与调用解耦，便于后期的修改和维护。

    public class DynamicHelloProxy implements InvocationHandler {
        private Object object;

        public DynamicHelloProxy(Object object) {
            this.object = object;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("Proxy do before invoking " + method.getName());
            method.invoke(object, args);
            System.out.println("Proxy do after invoking " + method.getName());
            return null;
        }
    }

调用：

    public class Client {
        public static void main(String[] args) {
            DynamicHelloProxy proxy = new DynamicHelloProxy(new Hello());
            HelloInterface hello = (HelloInterface) Proxy.newProxyInstance(
                    proxy.getClass().getClassLoader(),
                    Hello.class.getInterfaces(),
                    proxy);
            hello.sayHello();
        }
    }

    /** 
    输出结果：
        Proxy do before invoking sayHello
        Hello~
        Proxy do after invoking sayHello
    */

在动态代理中，核心是InvocationHandler。每一个代理的实例都会有一个关联的调用处理程序(InvocationHandler)。对待代理实例进行调用时，将对方法的调用进行编码并指派到它的调用处理器(InvocationHandler)的invoke方法。所以对代理对象实例方法的调用都是通过InvocationHandler中的invoke方法来完成的，而invoke方法会根据传入的代理对象、方法名称以及参数决定调用代理的哪个方法。

上述语法在1.7里也是适用的。

上面说了优点，下面比较一下和静态代理的不同。

- 在静态代理中我们需要对哪个接口和哪个被代理类创建代理类，所以我们在编译前就需要代理类实现与被代理类相同的接口，并且直接在实现的方法中调用被代理类相应的方法；
- 但是动态代理则不同，我们不知道要针对哪个接口、哪个被代理类创建代理类，因为它是在运行时被创建的。

或总结为：JDK静态代理是通过直接编码创建的，而JDK动态代理是利用反射机制在运行时创建代理类的。

### 2.2 原理
代理类生成是通过Proxy类中的newProxyInstance来完成。

    @CallerSensitive
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            //在这里对某些安全权限进行检查，确保我们有权限对预期的被代理类进行代理
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * Look up or generate the designated proxy class. 查找或生成代理类
         */
        Class<?> cl = getProxyClass0(loader, intfs);

        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }

            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            //假如代理类的构造函数是private的，就使用反射来set accessible
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }

Class<?> cl = getProxyClass0(loader, intfs); 这里面的实现，还会有个proxyClassCache缓存，如果缓存命中，直接返回代理类；否则通过ProxyClassFactory来创建。

后续内容就不深究了。 （对，我就是不求甚解！）

## 三、CGLIB 动态代理
在Spring AOP中，通常会用CGLIB来生成AopProxy对象。

### 3.1 例子和分析
测试前先添加cglib依赖：

    compile 'cglib:cglib:3.1'

实现MethodInterceptor接口生成方法拦截器:

    import org.springframework.cglib.proxy.MethodInterceptor;
    import org.springframework.cglib.proxy.MethodProxy;

    public class HelloMethodInterceptor implements MethodInterceptor {
        @Override
        public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
            System.out.println("Proxy do before invoking "+ method.getName());
            Object object= methodProxy.invokeSuper(o, objects);
            System.out.println("Proxy do after invoking "+ method.getName());
            return object;
        }
    }

生成代理类对象并打印在代理类对象调用方法之后的执行结果

    import org.springframework.cglib.proxy.Enhancer;

    public class Client {
        public static void main(String[] args) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(Hello.class);
            enhancer.setCallback(new HelloMethodInterceptor());
            HelloInterface hello = (HelloInterface) enhancer.create();
            hello.sayHello();
        }
    }

    /**
    输出结果：
        Proxy do before invoking sayHello
        Hello~
        Proxy do after invoking sayHello
     */

JDK代理要求被代理的类必须实现接口，有很强的局限性。而CGLIB动态代理则没有此类强制性要求。简单的说，CGLIB会让生成的代理类继承被代理类，并在代理类中对代理方法进行强化处理(前置处理、后置处理等)。

在CGLIB底层，其实是借助了ASM这个非常强大的Java字节码生成框架。

从上例中我们看到，代理类对象是由Enhancer类创建的。Enhancer是CGLIB的字节码增强器，可以很方便的对类进行拓展。

### 3.2 CGLIB 创建代理对象的步骤

CGLIB创建代理对象的几个步骤:

- 生成代理类的二进制字节码文件；
- 加载二进制字节码，生成Class对象( 例如使用Class.forName()方法 )；
- 通过反射机制获得实例构造，并创建代理类对象

**在JDK代理中方法的调用是通过反射来完成的。但是在CGLIB中，方法的调用并不是通过反射来完成的，而是直接对方法进行调用**：FastClass对Class对象进行特别的处理，比如将会用数组保存method的引用，每次调用方法的时候都是通过一个index下标来保持对方法的引用。

## 四、代理方式对比

下面给出三种代理方式之间对比

| 代理方式 | 实现 | 优点 | 缺点 | 特点|
|--|--|--|--|--|
| JDK静态代理 |代理类与委托类实现同一接口，并且在代理类中需要硬编码接口|实现简单，容易理解|代理类需要硬编码接口，在实际应用中可能会导致重复编码，浪费存储空间并且效率很低| -|
| JDK动态代理 |代理类与委托类实现同一接口，主要是通过代理类实现InvocationHandler并重写invoke方法来进行动态代理的，在invoke方法中将对方法进行增强处理|不需要硬编码接口，代码复用率高|只能够代理实现了接口的委托类|底层使用反射机制进行方法的调用|
|CGLIB动态代理|代理类将委托类作为自己的父类并为其中的非final委托方法创建两个方法，一个是与委托方法签名相同的方法，它在方法中会通过super调用委托方法；另一个是代理类独有的方法。在代理方法中，它会判断是否存在实现了MethodInterceptor接口的对象，若存在则将调用intercept方法对委托方法进行代理|可以在运行时对类或者是接口进行增强操作，且委托类无需实现接口|不能对final类以及final方法进行代理|底层将方法全部存入一个数组中，通过数组索引直接进行方法调用|


## REF
> [深入理解JDK动态代理机制](https://www.jianshu.com/p/471c80a7e831)  
> [深入理解CGLIB动态代理机制](https://www.jianshu.com/p/9a61af393e41?from=timeline&isappinstalled=0)
