---
layout: post
title: Spring Security
category: spring
comments: true
---

## 一、核心组件
介绍一些在Spring Security中常见且核心的Java类。

### 1.1 SecurityContextHolder

SecurityContextHolder用于存储安全上下文（security context）的信息。当前操作的用户是谁，该用户是否已经被认证，他拥有哪些角色权限…这些都被保存在SecurityContextHolder中。
**SecurityContextHolder默认使用ThreadLocal 策略来存储认证信息。看到ThreadLocal也就意味着，这是一种与线程绑定的策略。**Spring Security在用户登录时自动绑定认证信息到当前线程，在用户退出时，自动清除当前线程的认证信息。

### 1.2 Authentication 身份信息的抽象
先看看这个接口的源码长什么样：

    package org.springframework.security.core;// <1>
    public interface Authentication extends Principal, Serializable { // <1>
        Collection<? extends GrantedAuthority> getAuthorities(); // <2>
        Object getCredentials();// <2>
        Object getDetails();// <2>
        Object getPrincipal();// <2>
        boolean isAuthenticated();// <2>
        void setAuthenticated(boolean var1) throws IllegalArgumentException;
    }

<1> Authentication是spring security包中的接口，直接继承自Principal类，而Principal是位于java.security包中的。可以见得，Authentication在spring security中是最高级别的身份/认证的抽象。

<2> 由这个顶级接口，我们可以得到用户拥有的权限信息列表，密码，用户细节信息，用户身份信息，认证信息。
还记得1.1节中，authentication.getPrincipal()返回了一个Object，我们将Principal强转成了Spring Security中最常用的UserDetails，这在Spring Security中非常常见，接口返回Object，使用instanceof判断类型，强转成对应的具体实现类。接口详细解读如下：

- getAuthorities()，权限信息列表，默认是GrantedAuthority接口的一些实现类，通常是代表权限信息的一系列字符串。
- getCredentials()，密码信息，用户输入的密码字符串，在认证过后通常会被移除，用于保障安全。
- getDetails()，细节信息，web应用中的实现接口通常为 WebAuthenticationDetails，它记录了访问者的ip地址和sessionId的值。
- getPrincipal()，最重要的身份信息，大部分情况下返回的是UserDetails接口的实现类，也是框架中的常用接口之一。UserDetails接口将会在下面的小节重点介绍。

#### Spring Security是如何完成身份认证的？
1 用户名和密码被过滤器获取到，封装成Authentication,通常情况下是UsernamePasswordAuthenticationToken这个实现类。

2 AuthenticationManager 身份管理器负责验证这个Authentication

3 认证成功后，AuthenticationManager身份管理器返回一个被填充满了信息的（包括上面提到的权限信息，身份信息，细节信息，但密码通常会被移除）Authentication实例。

4 SecurityContextHolder安全上下文容器将第3步填充了信息的Authentication，通过SecurityContextHolder.getContext().setAuthentication(…)方法，设置到其中。

### 1.3 AuthenticationManager 身份认证器
初次接触Spring Security的朋友相信会被AuthenticationManager，ProviderManager，AuthenticationProvider …这么多相似的Spring认证类搞得晕头转向，但只要稍微梳理一下就可以理解清楚它们的联系和设计者的用意。

AuthenticationManager（接口）是认证相关的核心接口，也是发起认证的出发点，因为在实际需求中，我们可能会允许用户使用用户名+密码登录，同时允许用户使用邮箱+密码，手机号码+密码登录，甚至，可能允许用户使用指纹登录，所以说AuthenticationManager一般不直接认证，AuthenticationManager接口的常用实现类ProviderManager内部会维护一个List<AuthenticationProvider>列表，存放多种认证方式，实际上这是委托者模式的应用（Delegate）。

也就是说，核心的认证入口始终只有一个：AuthenticationManager，不同的认证方式：用户名+密码（UsernamePasswordAuthenticationToken），邮箱+密码，手机号码+密码登录则对应了三个AuthenticationProvider。这样一来就好理解多了吧？在默认策略下，只需要通过一个AuthenticationProvider的认证，即可被认为是登录成功。

### 1.4 DaoAuthenticationProvider
AuthenticationProvider最常用的一个实现便是DaoAuthenticationProvider。顾名思义，Dao正是数据访问层的缩写，也暗示了这个身份认证器的实现思路。

在Spring Security中。提交的用户名和密码，被封装成了UsernamePasswordAuthenticationToken，而根据用户名加载用户的任务则是交给了UserDetailsService，在DaoAuthenticationProvider中，对应的方法便是retrieveUser，虽然有两个参数，但是retrieveUser只有第一个参数起主要作用，返回一个UserDetails。

还需要完成UsernamePasswordAuthenticationToken和UserDetails密码的比对，这便是交给additionalAuthenticationChecks方法完成的，如果这个void方法没有抛异常，则认为比对成功。比对密码的过程，用到了PasswordEncoder和SaltSource，它们为保障安全而设计，都是比较基础的概念。

一个md5密码加密器：

    public class Md5PasswordEncoder implements PasswordEncoder {
        @Override
        public String encode(CharSequence rawPassword) {
            return DigestUtils.md5Hex(rawPassword.toString());
        }

        @Override
        public boolean matches(CharSequence rawPassword, String encodedPassword) {
            return StringUtils.equals(encodedPassword, encode(rawPassword));
        }
    }

### 1.5 UserDetails与UserDetailsService
上面不断提到了UserDetails这个接口，它代表了最详细的用户信息，这个接口涵盖了一些必要的用户信息字段，具体的实现类对它进行了扩展。

    public interface UserDetails extends Serializable {
       Collection<? extends GrantedAuthority> getAuthorities();
       String getPassword();
       String getUsername();
       boolean isAccountNonExpired();
       boolean isAccountNonLocked();
       boolean isCredentialsNonExpired();
       boolean isEnabled();
    }

它和Authentication接口很类似，比如它们都拥有username，authorities，区分他们也是本文的重点内容之一。Authentication的getCredentials()与UserDetails中的getPassword()需要被区分对待，前者是用户提交的密码凭证，后者是用户正确的密码，认证器其实就是对这两者的比对。Authentication中的getAuthorities()实际是由UserDetails的getAuthorities()传递而形成的。还记得Authentication接口中的getUserDetails()方法吗？其中的UserDetails用户详细信息便是经过了AuthenticationProvider之后被填充的。

    public interface UserDetailsService {
       UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
    }

UserDetailsService和AuthenticationProvider两者的职责常常被人们搞混，关于他们的问题在文档的FAQ和issues中屡见不鲜。记住一点即可：UserDetailsService只负责从特定的地方（通常是数据库）加载用户信息，仅此而已。记住这一点，可以避免走很多弯路。

UserDetailsService常见的实现类有JdbcDaoImpl，InMemoryUserDetailsManager，前者从数据库加载用户，后者从内存中加载用户，也可以自己实现UserDetailsService，通常这更加灵活。

### 1.6 架构概览图

![security](/images/201812/spring-security.png)

这张概览图能看出一些关键类直接的关系，方便架构设计。

## REF
> [Spring Security ( 一 ) ：架构概述](http://www.importnew.com/26712.html)
