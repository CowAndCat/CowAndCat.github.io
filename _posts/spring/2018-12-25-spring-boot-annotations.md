---
layout: post
title: Spring Boot - Annotations
category: spring
comments: true
---

## 1. @ConditionalOnProperty

配置Spring Boot通过@ConditionalOnProperty来控制Configuration是否生效。

具体操作是通过其两个属性name以及havingValue来实现的，其中name用来从application.properties中读取某个属性值，如果该值为空，则返回false;如果值不为空，则将该值与havingValue指定的值进行比较，如果一样则返回true;否则返回false。如果返回值为false，则该configuration不生效；为true则生效。

代码：

    @Configuration
    //如果synchronize在配置文件中并且值为true
    @ConditionalOnProperty(name = "synchronize", havingValue = "true")
    public class SecondDatasourceConfig {

        @Bean(name = "SecondDataSource")
        @Qualifier("SecondDataSource")
        @ConfigurationProperties(prefix = "spring.second.datasource")
        public DataSource jwcDataSource() {
            return DataSourceBuilder.create().build();
        }
    }

和

    @ConditionalOnProperty(prefix = "auth", name = "authType", havingValue = "EXTERNAL")
    @EnableWebSecurity
    public class ExternalWebSecurityConfigurer extends AdnWebSecurityConfigurer {
        ...
    }

## 2. @Scope

用来标识bean的作用域，取值有：singleton、prototype、request、session、global session，均为字符串。一般prototype也称non-singletion。Spring2.0以后，增加了session、request、global session三种专用于Web应用程序上下文的Bean。

- singletion: 当一个bean的作用域设置为singleton, 那么Spring IOC容器中只会存在一个共享的bean实例，并且所有对bean的请求，只要id与该bean定义相匹配，则只会返回bean的同一实例。

- prototype: 用prototype作用域部署的bean，每一次请求（将其注入到另一个bean中，或者以程序的方式调用容器的 getBean()方法）**都会产生一个新的bean实例，相当与一个new的操作。Spring不能对一个prototype bean的整个生命周期负责**，容器在初始化、配置、装饰或者是装配完一个prototype实例后，将它交给客户端，随后就对该prototype实例不闻不问了。不管何种作用域，容器都会调用所有对象的初始化生命周期回调方法，而对prototype而言，任何配置好的析构生命周期回调方法都将不会被调用。

- request表示该针对每一次HTTP请求都会产生一个新的bean，同时该bean仅在当前HTTP request内有效。

- session作用域表示该针对每一次HTTP请求都会产生一个新的bean，同时该bean仅在当前HTTP session内有效。

- global session作用域类似于标准的HTTP Session作用域，不过它仅仅在基于portlet的web应用中才有意义。Portlet规范定义了全局Session的概念，它被所有构成某个 portlet web应用的各种不同的portlet所共享。在global session作用域中定义的bean被限定于全局portlet Session的生命周期范围内。如果你在web中使用global session作用域来标识bean，那么web会自动当成session类型来使用。

    配置示例：

        <bean id="role" class="spring.chapter2.maryGame.Role" scope="prototype"/>
        或
        @Component
        @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
        public class ExecutorServiceProxy {...}

## 3. @Bean(destroyMethod = "shutdown")

使用javaConfig配置的bean，如果存在close或者shutdown方法，则在bean销毁时会自动执行该方法，如果你不想执行该方法，则添加@Bean(destroyMethod="")来防止触发销毁方法.

例如：

    <bean id="xxx" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">

BasicDataSource提供了close()方法关闭数据源，所以必须设定destroy-method=”close”属性， 以便Spring容器关闭时，数据源能够正常关闭；销毁方法调用close(),是将连接关闭，并不是真正的把资源销毁。

还可以理解成：当数据库连接不使用的时候,就把该连接重新放到数据池中,方便下次使用调用.

## 4.  @PostConstruct 和 @PreConstruct
这两个注解被用来修饰一个非静态的void()方法.而且这个方法不能有抛出异常声明。

    @PostConstruct                                 //方式1
    public void someMethod(){
        ...
    }

    public @PostConstruct void someMethod(){        //方式2
        ...  
    }

@PostConstruct说明:  
被@PostConstruct修饰的方法会在服务器加载Servlet的时候运行，并且只会被服务器调用一次，类似于Servlet的init()方法。被@PostConstruct修饰的方法会在构造函数之后，init()方法之前运行。

@PreDestroy说明:  
被@PreDestroy修饰的方法会在服务器卸载Servlet的时候运行，并且只会被服务器调用一次，类似于Servlet的destroy()方法。被@PreDestroy修饰的方法会在destroy()方法之后运行，在Servlet被彻底卸载之前。

## 5. `@Pointcut("within(com.baidu.bce..*Controller) && @target(classRequestMapping) && @annotation(methodRequestMapping)")`

    @Pointcut("within(com.baidu.bce..*Controller) && @target(classRequestMapping) && @annotation(methodRequestMapping)")
    public void requestMappingPointcut(RequestMapping classRequestMapping, RequestMapping methodRequestMapping) {
    }

这是一个使用 AspectJ 风格切面的配置，使得 spring 的切面配置大大简化。参考：[详解Spring 框架中切入点 pointcut 表达式的常用写法](https://www.jb51.net/article/110461.htm)

Pointcut的定义包括两个部分：Pointcut表示式(expression)和Pointcut签名(signature)

`within(com.baidu.bce..*Controller)`: 表示在`com.baidu.bce`包或其子包中的任意连接点的所有以Controller结尾的类  

`target(SomeType)`：when the target object is of type SomeType (当对象是某种类型时)

@within和@target针对类的注解,@annotation是针对方法的注解。

## 6. @Around("requestMappingPointcut(classRequestMapping, methodRequestMapping)")

@Before/@After 是在所拦截方法执行之前/之后执行一段逻辑。@Around是可以同时在所拦截方法的前后执行一段逻辑。

    @Aspect
    @Component
    public class LogIntercept {

        @Pointcut("execution(public * com.itsoft.action..*.*(..))")
        public void recordLog(){}

        @Before("recordLog()")
        public void before() {
            this.printLog("已经记录下操作日志@Before 方法执行前");
        }

        @Around("recordLog()")
        public void around(ProceedingJoinPoint pjp) throws Throwable{
            this.printLog("已经记录下操作日志@Around 方法执行前");
            pjp.proceed();
            this.printLog("已经记录下操作日志@Around 方法执行后");
        }

        @After("recordLog()")
        public void after() {
            this.printLog("已经记录下操作日志@After 方法执行后");
        }

        private void printLog(String str){
            System.out.println(str);
        }
    }

虽然Around功能强大，但通常需要在线程安全的环境下使用。因此，如果使用普通的Before、After、AfterReturing增强方法就可以解决的事情，就没有必要使用Around增强处理了。

一个用@Pointcut和@Around的完整例子：

    @Aspect
    @Component
    public class LatencyAspect {

        @Pointcut("@annotation(latency)")
        public void latencyPointcut(Latency latency) {
        }

        @Around("latencyPointcut(latency)")
        public Object around(ProceedingJoinPoint joinPoint, Latency latency) throws Throwable {
            long start = System.currentTimeMillis();
            try {
                Object object = joinPoint.proceed(); 
                long diff = System.currentTimeMillis() - start;
                LatencyRecorderMap.get(String.format("latency_%s", latency.value())).supply(diff);
                return object;
            } catch (Throwable ex) {
                long diff = System.currentTimeMillis() - start;
                LatencyRecorderMap.get(String.format("latency_throw_%s", latency.value())).supply(diff);
                throw ex;
            }
        }
    }

@Lantency仅仅是个包含value的注解

    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    public @interface Latency {
        String value();
    }

使用：

    @Latency("vaas_image_classify")
    private ImageClassifyResponse httpRequestEach(String url, ImageClassifyRequest request) throws IOException {
        String response = BceInternalClient.request(url).post(Entity.json(request), String.class);
        return JsonUtils.fromJsonString(response, ImageClassifyResponse.class);
    }

有时间看看AspectJ

## 7. @Cacheable(value = "endpoint_urls")

[《Spring缓存注解@Cacheable、@CacheEvict、@CachePut使用》](https://www.cnblogs.com/fashflying/p/6908028.html)

    @Cacheable(value={"users"}, key="#user.id", condition="#user.id%2==0")
    public User find(User user) {
        System.out.println("find user by user " + user);
        return user;
    }

当前方法的返回值会被缓存在users这个Cache上，以user.id作为key来区分，只缓存id为偶数的那些数据。（注意value、key、condition的使用）

value表示清除操作是发生在哪些Cache上的（对应Cache的名称）；key表示需要清除的是哪个key，如未指定则会使用默认策略生成的key；condition表示清除操作发生的条件。

在支持Spring Cache的环境下，对于使用@Cacheable标注的方法，Spring在每次执行前都会检查Cache中是否存在相同key的缓存元素，如果存在就不再执行该方法，而是直接从缓存中获取结果进行返回，否则才会执行并将返回结果存入指定的缓存中。

@CachePut也可以声明一个方法支持缓存功能。与@Cacheable不同的是使用@CachePut标注的方法在执行前不会去检查缓存中是否存在之前执行过的结果，而是**每次都会执行该方法，并将执行结果以键值对的形式存入指定的缓存中**。

    @CachePut("users")//每次都会执行方法，并将结果存入指定的缓存中
    public User find(Integer id) {
        return null;
    }

@CachePut也可以标注在类上和方法上。使用@CachePut时我们可以指定的属性跟@Cacheable是一样的。

    @CacheEvict(value = "cache1", key = "#endpoint")
    void func(){
    }

@CacheEvict是用来标注在需要清除缓存元素的方法或类上的。@CacheEvict可以指定的属性有value、key、condition、allEntries和beforeInvocation。

allEntries是boolean类型，表示是否需要清除缓存中的所有元素。默认为false，表示不需要。

beforeInvocation清除操作默认是在对应方法成功执行之后触发的，即方法如果因为抛出异常而未能成功返回时也不会触发清除操作。使用beforeInvocation可以改变触发清除操作的时间，当我们指定该属性值为true时，Spring会在调用该方法之前清除缓存中的指定元素。

@Caching注解可以让我们在一个方法或者类上同时指定多个Spring Cache相关的注解。其拥有三个属性：cacheable、put和evict，分别用于指定@Cacheable、@CachePut和@CacheEvict。
