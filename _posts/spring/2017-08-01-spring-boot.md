---
layout: post
title: Spring Boot for Starters
category: spring
comments: false
---
>[https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)

## 置前内容
 [Spring Initializr-在线生成项目](https://start.spring.io/)：[https://start.spring.io/](https://start.spring.io/)

能快速生成项目框架！

# 一、 Step up
## 1. Spring Boot 快速起步

首先到上面网站上构建一个gradle project，下载解压后，用idea打开，打开过程中注意勾选 auto import.

之后新建一个类：

    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RestController;

    /**
     * Example.
     *
     * @author haluomao
     */
    @RestController
    @EnableAutoConfiguration
    public class Example {
        @RequestMapping("/")
        String home() {
            return "Hello World!";
        }
        public static void main(String[] args) throws Exception {
            SpringApplication.run(Example.class, args);
        }
    }

在 build.gradle 中增加依赖，如下：

    dependencies {
        compile('org.springframework.boot:spring-boot-starter')
        compile('org.springframework.boot:spring-boot-starter-web')
        testCompile('org.springframework.boot:spring-boot-starter-test')
    }

加载完毕后，运行Example.java， 启动后访问http://localhost:8080/ 就能看到『 Hello World!』

## 2. 一些注解
基础的注解不再赘述，可参考笔者以前的文章。

- @EnableAutoConfiguration 属于class-level，这个注解会让Boot基于现有的jar依赖去『猜』你想怎么配置Spring. 上面因为spring-boot-starter-web 添加了Tomcat和Spring MVC, 这个自动配置就推断出你要开发一个web app并启动spring. （虽然是为starter准备的，你也可以自行配置依赖，但还是很建议使用） 

- @EnableAutoConfiguration(exclude={DataSourceAutoConfiguration.class}) exclude属性可以去除一些不需自动配置的类。

- @Configuration 用来标记配置类。 除了这个Annotation,还有@Import, @ComponentScan. 虽然通过xml也能够定义配置，但是spring boot还是推荐使用java-based configuration.(即将配置写在java代码里)

- @SpringBootApplication 这个标注用于简化一组标注（因为它们经常一起出现）：@Configuration, @EnableAutoConfiguration and @ComponentScan，是等价的。
- @Autowired 一般用于构造函数，来自动注入beans参数.

- @ConfigurationProperties 利用内置的配置读取器来加载配置，加载的内容会是Properties. 被加载的类可以作为Component来使用。如@ConfigurationProperties(prefix="foo")
- @Value 相对于@ConfigurationProperties，不是Relaxed binding, 不支持Meta-data support(类型检查)，但支持 SpEL expression. 建议用上者。
- @Profile 用于隔离不同环境下的配置，比如@Profile("production")只会在配置为产品的时候才会生效。
- @Transactional 来标记事务体。

## 3. 可执行的jar

Java 没有标准的方式去加载内嵌的jar包（jar里面的jar），一种方式是用『ubers』jars,即将jars里的类都拆解到一个archive中，但这样就无法再看出应用使用了哪些library，而且还会存在同名的冲突。

Boot 采用另一方式：将应用的类放在内嵌的 BOOT-INF/classes 下，将依赖放在BOOT-INF/lib下。

打包方式：

    ./gradlew jar

生成的jar文件在build/libs下面。

运行： java -jar JAR_FILE  (--debug)  
或者： gradle bootRun

## 4. Starter

Spring Boot 提供了一些app Starter来让新手可以跳过繁琐的dependency配置，来更快地上路。  
这些starter其实是一个个依赖集合，比如spring-boot-starter集中了yaml/Core/logging等依赖。

可以参考： https://github.com/spring-projects/spring-boot/tree/v1.5.6.RELEASE

## 5. Spring Beans and dependency injection

Application components (as beans): @Component, @Service, @Repository, @Controller etc.

DI: 在构造函数上用@Autowired来自动注入beans.

## 6. Hot swapping 热部署

因为Boot应用只是简单的java应用，所以它支持只在bytecode层面进行替换的热部署。解决方案有： 

- JRebel （https://zeroturnaround.com/software/jrebel/ not free）
- Spring Loaded (https://github.com/spring-projects/spring-loaded)
- spring-boot-devtools (一个dependency module, 支持快速重启.)

# 二、Dive in

## 1. 自定义开机画面
可以在classpath添加如下内容来定制开机画面（主要是在启动后的控制台输出）：

- banner.txt 里面的内容会照搬显示到控制台。（可以放在resources下）  
在程序里配置：
    
        SpringApplication.setBanner(...)
    实现org.springframework.boot.Banner interface的printBanner()接口。

- banner.gif/banner.jpg/banner.png 图片会转成ASCII然后显示。（很有意思的）

- 可以在application.properties里定制开机画面，比如可以使用在线的图资源：banner.image.location = https://avatars2.githubusercontent.com/u/1695304?v=4&s=460
然后你就会发现，开机画面变成了一张文字图。

- banner.txt 里面还可以设置一些placeholder，比如 ${spring-boot.version}，在启动的时候会被替换成 1.5.6.RELEASE，还可以是：
 ${application.version} 表示版本， ${application.title}表示应用名，等等，这些会从MANIFEST.MF里获取。

- spring.main.banner-mode 取值：on System.out (*console*), using the configured logger (*log*) or not at all (*off*).  
如果是在YAML里，配置如下：

        spring:
            main:
                banner-mode: "off"
如果是在代码里：

        app.setBannerMode(Banner.Mode.OFF);

## 2. App events and listeners

注册Application级别Event的Listener方式：  

    SpringApplication.addListeners(...) or SpringApplicationBuilder.listeners(...) or
    org.springframework.context.ApplicationListener=MyLister (in META-INF/spring.factories)

(注意，不能使用@bean来注册，因为有些事件会在ApplicationContext创建之前就会被触发)

一些Event(按发生顺序排列）:

1. An **ApplicationStartingEvent** is sent at the start of a run, but before any processing except the registration of listeners and initializers.
2. An **ApplicationEnvironmentPreparedEvent** is sent when the Environment to be used in the context is known, but before the context is created.
3. An **ApplicationPreparedEvent** is sent just before the refresh is started, but after bean definitions have been loaded.
4. An **ApplicationReadyEvent** is sent after the refresh and any related callbacks have been processed to indicate the application is ready to service requests.
5. An **ApplicationFailedEvent** is sent if there is an exception on startup.

You often won’t need to use application events, but it can be handy to know that they exist. Internally, Spring Boot uses events to handle a variety of tasks.

## 3. Configuration
1. 配置方式有四种： properties files, YAML files, environment variables and command-line arguments (最后一个算是运行时的配置)

2. 除了properties文件，还可以通过profile-specific 属性来制定配置，比如 spring.profiles.active 属性。

    spring:
      profiles:
        active: production

**YAML**  

3. YAML is a superset of JSON, and as such is a very convenient format for specifying hierarchical configuration data. （类似JSON，非常适用于层次结构。）

    4. Spring Framework提供两种方式去加载YAML文件：YamlPropertiesFactoryBean将配置加载成Properties, YamlMapFactoryBeanwill load YAML as a Map.

    YAML 列表配置：

        my:
            servers:
                - dev.bar.com
                - foo.bar.com
    会转换成：

        my.servers[0]=dev.bar.com
        my.servers[1]=foo.bar.com

5. 绑定属性可以使用注解 @ConfigurationProperties，例如绑定上文中的配置（而且还不用写setter）：

        @ConfigurationProperties(prefix="my")
        public class Config {
            private List<String> servers = new ArrayList<String>();
            public List<String> getServers() { 
                return this.servers;
            } 
        }

6. 可以将多个 profile-specific YAML文件写到一个文件里，使用 spring.profiles 来标明使用哪个文档。（当有敏感信息的时候，不要这样做）

        server:
            address: 192.168.1.100
        ---
        spring:
            profiles: development
        server:
            address: 127.0.0.1
        ---
        spring:
            profiles: production
        server:
            address: 192.168.1.120

7. 还可以添加其他Profile,当配置为prod的时候，也会激活proddb和prodmq配置：

        spring.profiles: prod
        spring.profiles.include:
          - proddb
          - prodmq

## 4. Logging (略) #P74 (88)

## 5. Developing web applications 

### 5.1 Static Content

默认的资源路径：/static 或 /public 或 /resources

修改默认的资源路径方法：

    spring.mvc.static-path-pattern=/resources/**
或 spring.resources.static-locations （后接一个dir列表）

注意：不要放在 src/main/webapp 下，这样只会在war包里有效，在jar里面没用。

Custom Favicon： 只需将 favicon.ico 放在配置的static路径下。

### 5.2 Security
OAuth2 (Authorization Server and grant access tokens)/ Single Sign On /Actuator Security

感兴趣再深入。

### 5.3 SQL DB
支持的内存型数据库：H2, HSQL and Derby.   
不需要指定任何连接URL，只要include一个dependency. 例如：

    <dependency>
        <groupId>org.hsqldb</groupId>
        <artifactId>hsqldb</artifactId>
        <scope>runtime</scope>
    </dependency>

对于非嵌入式的数据库，DataSource Configuration:

    spring.datasource.url=jdbc:mysql://localhost/test spring.datasource.username=dbuser
    spring.datasource.password=dbpass spring.datasource.driver-class-name=com.mysql.jdbc.Driver
driver-class-name 都可以不必刻意指定，以为boot能根据url来deduce it.

Pooling DataSource (jdbc的实现方式) 选择规则： Tomcat pooling DataSource > HikariCP > Commons DBCP

> 扫盲  
> 
- Spring Data **JPA** repositories are interfaces that you can define to access data.
- **Java Object Oriented Querying (jOOQ)** is a popular product from Data Geekery which generates Java code from your database, and lets you build type safe SQL queries through its fluent API. 

歪楼：Mybatis很好，我一直用它。

### 5.4 Working with NoSQL tech
including: MongoDB, Neo4J, Elasticsearch, Solr, Redis, Gemfire, Cassandra, Couchbase and LDAP. 

Connecting to Redis:

    @Component
    public class MyBean {
        private StringRedisTemplate template;
        @Autowired
        public MyBean(StringRedisTemplate template) { 
            this.template = template;
        }
        // ...
    }

### 5.5 Caching
在Boot里使用Cache是一件很容易的事情，可以通过@EnableCaching来开启缓存支持，通过@Cacheable来使用。

使用示例：

    import org.springframework.cache.annotation.Cacheable;
    import org.springframework.stereotype.Component;
    @Component
    public class MathService {

        @Cacheable("piDecimals")
        public int computePiDecimal(int i) { 
            // ...
        } 
    }
当调用computePiDecimal方法的时候，如果在piDecimals缓存中有一个叫i的entry，则直接返回其对应的值，否则执行函数并将值存入到cache中。

若未制定cache provider，boot会用基于内存的concurrent map来实现缓存。  
支持的Providers (按优先级使用）: Generic、JCache、EhCache 2.x、Hazelcast、Infinispan、Couchbase、Redis、Caffeine、Simple

使用spring.cache.type能强制指定provider, 但是建议在特定的场景下关闭，比如在测试的时候。

### 5.6 Messaging

- JMS API，支持的实现有：ActiveMQ, Artemis.
- Spring AMQP（Advanced Message Queuing Protocol）：RabbitMQ.
- 还支持STOMP， Apache Kafka.

> 扫盲
> 
- The Advanced Message Queuing Protocol (AMQP) is a platform-neutral, wire-level protocol for message-oriented middleware.
- RabbitMQ is a lightweight, reliable, scalable and portable message broker based on the AMQP protocol. Spring uses RabbitMQ to communicate using the AMQP protocol.

### 5.7 Validation

使用示例：

    @Service
    @Validated
    public class MyBean {
        public Archive findByCodeAndAuthor(@Size(min = 8, max = 10) String code, Author author) {
        //... 
        }
    }

### 5.8 Distributed Transactions with JTA
分布式事务！

Spring Boot supports distributed JTA transactions across multiple XA resources using either an Atomikos or Bitronix embedded transaction manager.  
JTA transactions are also supported when deploying to a suitable Java EE Application Server.

事务管理器： Atomikos、Bitronix、Narayana

> Narayana is popular open source JTA transaction manager implementation supported by JBoss. You can use the spring-boot-starter-jta-narayana starter to add the appropriate Narayana dependencies to your project. As with Atomikos and Bitronix, Spring Boot will automatically configure Narayana and post-process your beans to ensure that startup and shutdown ordering is correct.

### 5.9 Testing
- By default, Spring Boot uses Mockito 1.x.

- >Hamcrest — A library of matcher objects (also known as constraints or predicates).

- If your test is @Transactional, it will rollback the transaction at the end of each test method by default.

- @ContextConfiguration(classes=...) in order to specify which Spring @Configuration to load.

# 三、小结

### 1. Spring Boot的四大神器（主要功能）

- 自动配置(auto-configuration)  
一项简化配置的功能，比如在classpath中发现有spring security的jar包，则自动创建相关的bean等
- starters(简化依赖)  
    这个比较关键，方便spring去集成各类组件，比如redis、mongodb等等。
    - core (security、aop)
    - web (web、websocket、ws、vaadin、rest、mobile)
    - template (freemarker、velocity、groovy templates、thymeleaf)
    - data (jdbc、jpa、mongodb、redis、gemfire、solr、elasticsearch)
    - database (h2、hsqldb、mysql、postgresql)
    - social (facebook、linkedin、twitter)
    - io (batch、integration、jms、amqp)
    - ops (actuator、remote shell)
- CLI (command-line interface),支持groovy开发
- [Actuator(对应用系统本身的自省功能)](https://segmentfault.com/a/1190000004318360?_ea=568366)  
比如查看系统运行了多少线程，gc的情况，运行的基本参数等等。

### 2. 阅读建议
细节比较多，笔者认为不必深入，用到的时候再研究，但是要先有认识，不然都意识不到何时该用什么。
