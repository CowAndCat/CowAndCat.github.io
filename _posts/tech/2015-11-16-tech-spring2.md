---
layout: post
title: spring配置
category: tech
comments: false
---
##一个标准的Spring配置文件：applicationContext.xml


	<?xml version="1.0" encoding="UTF-8"?>
	<beans xmlns="http://www.springframework.org/schema/beans"
	 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	 xmlns:context="http://www.springframework.org/schema/context"
	 xmlns:tx="http://www.springframework.org/schema/tx"
	 xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
	    http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-2.5.xsd
	    http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-2.5.xsd"
	 default-autowire="byName" default-lazy-init="true">
	
	
	 <!-- 配置数据源 -->
	 <bean id="dataSource"
	  class="org.springframework.jdbc.datasource.DriverManagerDataSource">
	  <property name="driverClassName">
	   <value>com.mysql.jdbc.Driver</value>
	  </property>
	  <property name="url">
	   <value>
	    jdbc:mysql://localhost/ssh?characterEncoding=utf-8
	   </value>
	  </property>
	  <property name="username">
	   <value>root</value>
	  </property>
	  <property name="password">
	   <value>123</value>
	  </property>
	 </bean>
	
	
	 <!--配置SessionFactory -->
	 <bean id="sessionFactory"
	  class="org.springframework.orm.hibernate3.LocalSessionFactoryBean">
	  <property name="dataSource">
	   <ref bean="dataSource" />
	  </property>
	  <property name="mappingResources">
	   <list>
	    <value>com/ssh/pojo/User.hbm.xml</value>
	   </list>
	  </property>
	  <property name="hibernateProperties">
	   <props>
	    <prop key="hibernate.show_sql">true</prop>
	   </props>
	  </property>
	 </bean>
	 
	 <!-- 事务管理 -->
	 <bean id="transactionManager"
	  class="org.springframework.orm.hibernate3.HibernateTransactionManager">
	  <property name="sessionFactory">
	   <ref bean="sessionFactory" />
	  </property>
	 </bean>
	 
	 <!-- hibernateTemplate -->
	 <bean id="hibernateTemplate"
	  class="org.springframework.orm.hibernate3.HibernateTemplate">
	  <property name="sessionFactory">
	   <ref bean="sessionFactory" />
	  </property>
	 </bean>
	
	
	 <!-- 配置数据持久层 -->
	 <bean id="userDao"
	  class="com.ssh.dao.impl.UserDaoImpl">
	  <property name="hibernateTemplate" ref="hibernateTemplate"></property>
	 </bean>
	 
	
	
	 <!-- 配置业务逻辑层 -->
	 <bean id="userService"
	  class="com.ssh.service.impl.UserServiceImpl">
	  <property name="userDao" ref="userDao"></property>
	 </bean>
	 
	
	
	 <!-- 配置控制层 -->
	 <bean id="UserAction"
	  class="com.ssh.action.UserAction"  scope="prototype">
	  <property name="userService" ref="userService"></property>
	 </bean>

	  <!-- 配置pojo -->
	 <bean id="User" class="com.ssh.pojo.User" scope="prototype"/>
	</beans>
