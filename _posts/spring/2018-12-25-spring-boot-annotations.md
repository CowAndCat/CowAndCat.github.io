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
