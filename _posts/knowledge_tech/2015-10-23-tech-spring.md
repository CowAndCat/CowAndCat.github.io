---
layout: post
title: 掌握spring
category: tech
comments: false
---
【喵体悟】学会用，对于一个要成为senior programmer的人是远远不够的，还要懂得其原理，知其所以然。

阅读《Spring技术内幕——深入解析Spring架构与设计原理》、
《精通Spring2.0》 罗时飞 2007（一看就知道是中国人写的书，利用一大堆名词和代码来凑数）、《Spring in Action（中文版）》 Craig Walls.etc. 

###1、Spring是什么？
Spring是一个轻量级的IoC和AOP容器框架。

- 轻量级：大小上和系统开支都是轻量级的，而且是非侵入式的。
- 反向控制：实现松耦合，使用IoC，对象是被动接受依赖而不是自己去找。
- 面向切片：将业务逻辑从系统服务中分离出来，达到内聚开发。系统对象只做它们该做的业务逻辑。
- 容器：Spring是一个容器，包含且管理系统对象的生命周期和配置。
- 框架：在Spring中，支持使用简单的组件配置组合成一个复杂的系统。并且提供了很多基础功能（事务管理、持久层集成等）。

缺点：业务功能依赖spring特有的功能，依赖与spring环境。

###2、Spring的主旨思想是什么？为什么会成功？
IoC 和 AOP

- 控制反转Ioc：BeanFactory使用工厂模式来实现，将系统的配置和依赖关系从代码中独立出来。
- AOP：一种编程技术，用于在提供中提升业务的分离。spring中面向切面变成的实现有两种方式，一种是动态代理，一种是CGLIB，动态代理必须要提供接口，而CGLIB实现是有继承。

###3、IOC容器
实现了对象创建责任的反转。

一个是BeanFactory、一个是ApplicationContext。

在spring中BeanFacotory是IoC容器的核心接口，负责实例化，定位，配置应用程序中的对象及建立这些对象间的依赖。

XmlBeanFacotory实现BeanFactory接口，通过获取xml配置文件数据，组成应用对象及对象间的依赖关系。

spring中有三种注入方式，一种是set注入，一种是接口注入，另一种是构造方法注入。

set注入和构造注入有时在做配置时比较麻烦。所以框架为了提高开发效率，提供自动装配功能，简化配置。  
Spring框架式默认不支持自动装配的，要想使用自动装配需要修改spring配置文件中<bean>标签的autowire属性

自动装配属性有6个值可选，分别代表不同的含义。（byName、byType、constructor、autodetect、no、default）

自动装配功能和手动装配要是同时使用，那么自动装配就不起作用。

###4、为什么springMVC和Mybatis逐渐流行起来了
Spring为Web系统提供了全功能的MVC框架。虽然Spring可以很容易地与其他MVC框架（如Struts)集成，但是Spring的MVC框架利用IoC将**控制逻辑和业务逻辑清晰地分离开**来。

SpringMVC的优点:

- 与Spring框架天生整合，无框架兼容问题
- 与Struts2相比安全性高
- 配置量小、开发效率高

MyBatis的优点：

- 不需要重新学习hibernate框架，在掌握sql的基础上就可以上手；
- 不需要配置实体类与数据表之间的映射关系；
- hibernate不能自己控制sql语句

###5、spring中的单元测试

测试类：最好单独建立项目，或者单独定义文件夹存储，需要继承junit.framework.TestCase
 
增加测试方法：  测试方法必须是public,不应该有返回值，方法名必须以test开头，无参数测试方法是有执行先后顺序，按照方法的定义先后顺序

多个测试方法对同一个业务方法进行测试，一般每个逻辑分支结构都有测试到。

	public class TestUser extends TestCase{ 
	    public void testUser_Success() throws Exception{ 
	       //准备数据
	       User action = new User();
	       action.setUsername("admin");
	
	       //调用被测试方法
	       String result = action.login();
	 
	       //判断测试是否通过
	       assertEquals("success",result); 
	    }
	}

setUp方法会在每一个测试方法前执行一次。tearDown方法会在每一个测试方法后执行一次。

###6、spring中的注解
注解Annotation，是一种类似注释的机制，在代码中添加注解可以在之后某时间使用这些信息。跟注释不同的是，注释是给我们看的，java虚拟机不会编译，注解也是不编译的，但是我们可以通过反射机制去读取注解中的信息。注解使用关键字@interface，继承java.lang.annotition.Annotition

使用注解编程，主要是为了替代xml文件，使开发更加快速。

在没有使用注解时，spring框架的配置文件applicationContext.xml文件中需要配置很多的<bean>标签，用来声明类对象。使用注解，则不必在配置文件中添加标签拉，对应的是在对应类的“注释”位置添加说明。

spring框架使用的是分层的注解。  
持久层：@Repository；  
服务层：@Service  
控制层：@Controller  

这三个层中的注解关键字都可以使用@Component来代替。 
 使用注解声明对象，默认情况下生成的id名称为类名称的首字母小写。

###7、从Spring环境中获取Action对象

	ServletContext application =request.getSession().getServletContext();
	ApplicationContextac = WebApplicationContextUtils.getWebApplicationContext(application);
	 
	UserAction useraction = (UserAction)ac.getBean("ua");//获取控制层对象
	
	response.setContentType("text/html;charset=GBK");//设置编码
	PrintWriter out =response.getWriter();
	
	//分别将三个层的对象打印出来。
	out.println("Action:"+userAction);
	out.println("Service:"+userAction.getUserService());
	out.println("Dao:"+userAction.getUserService().getUserDao());

###8、spring中的事务管理
Spring没有直接的事务管理，但是有很多的事务管理器可供选择。如：DataSourceTransactionManager、HibernateTransactionManager、JdoTransactionManager、JtaTransactionManager、PersistenceBrokerTransactionManager。

（JDO-Java data object/Jta-Java transaction API/OJB-Object Relational Bridge）

![1](/images/201511/transaction.jpg "Spring事务管理器")

例如，当持久化机制是Hibernate时，用HibernateTransactionManager来管理事务。

每种事务管理器都充当了对特定平台的事务实现的代理，这样你就只要和spring中的事务打交道。

（1）编程式实现事务

Spring提供两种**编程式事务支持**：直接使用PlatformTransactionManager实现和使用TransactionTemplate模板类，用于支持逻辑事务管理。如果采用编程式事务推荐使用TransactionTemplate模板类。

这里有详细的介绍：[Spring的事务 之 9.2 事务管理器 ——跟我学spring3](http://jinnianshilongnian.iteye.com/blog/1439900)

（2）Spring声明式事务 

在日常开发中，用的最多的就是声明式事务了，下面将介绍SpringJdbc的声明式事务的配置方法：

如：

	<!-- 配置数据源 -->      
	<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource" destroy-method="close">  
	    <!-- Connection Info -->  
	    <property name="driverClass" value="${db.driverClass}" />  
	    <property name="jdbcUrl" value="${db.url}" />  
	    <property name="user" value="${db.username}" />  
	    <property name="password" value="${db.password}" />  
	</bean>

####8.1 事务的属性

传播行为：定义了关于客户端和被调用方法的事务边界。

有7个，如：MANDATORY：指定当前方法必须加入当前事务环境，如果当前没有事务，就抛出异常。 REQUIRED：指定当前方法必需在事务环境中运行，如果当前有事务环境就加入当前正在执行的事务环境，如果当前没有事务，就新建一个事务。这是默认值。 

这种属性应该是加在方法上，要注意，有3个属性是会创建新的事务。


建议只在实现类或实现类的方法上使用@Transactional，而不要在接口上使用，这是因为如果使用JDK代理机制是没问题，因为其使用基于接口的代理；而使用CGLIB代理机制时就会遇到问题，因为其使用基于类的代理而不是接口，这是因为接口上的@Transactional注解是“不能继承的”。 

	@Transactional//放在这里表示所有方法都加入事务管理  
	public class AnnotationUserServiceImpl implements IUserService {  
	    private IUserDao userDao;  
	    private IAddressService addressService;  
	      
	    @Transactional(propagation=Propagation.REQUIRED, isolation=Isolation.READ_COMMITTED)  
	    public void save(UserModel user) {  
	        userDao.save(user);  
	        user.getAddress().setUserId(user.getId());  
	        addressService.save(user.getAddress());  
	    }  
	  
	    @Transactional(propagation=Propagation.REQUIRED, readOnly=true,  
	                   isolation=Isolation.READ_COMMITTED)  
	    public int countAll() {  
	        return userDao.countAll();  
	    }  
	    //setter...  
	}  

1. 通过方法名声明事务：顺序如下——传播行为，隔离级别，只读，回滚规则。  
2. 用元数据声明事务Jakarta Commons Attributes 和 JSR-175（两种元数据的实现）

###9、一个请求在spring mvc中的生命周期

![2](/images/201511/springLifecycle.gif "生命周期")

1、客户端发出请求第一个接受请求的组件是DispatcherServlet.(前端控制器模式)  
2、DispatcherServlet开始查询一个或多个HandlerMapping。一个HandlerMapping的工作主要是将URL映射到一个控制器对象。  
3、一旦DispatcherServlet找到了一个控制器对象，它将请求分派给这个控制器，让它根据设计的业务逻辑处理（3）这个请求。  
4、完成业务逻辑后，控制器返回一个ModelAndView（4）给DispatcherServlet。ModelAndView不是携带一个视图对象，就是携带一个视图对象的逻辑名.   
5、如果MedelAndView对象携带的是一个视图对象的逻辑名，DispactherServlet需要一个ViewResolver（5）来查找用于渲染回应的视图对象。最后，DispatcherServlet将请求分派给ModelAndView对象制定的视图对象（6）。视图对象负责渲染返回给客户的回应。ViewResolver将控制器和JSP结合起来。