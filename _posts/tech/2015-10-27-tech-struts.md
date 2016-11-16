---
layout: post
title: Struts
category: tech
comments: false
---
Struts是MVC框架的事实标准，已经存在多年了。

#2、 与Spring MVC不同的地方
- Struts中常用到的类是Action，但其是一个类而不是接口，所以后续所有处理请求的类都要继承Action。（Spring中提供了Controller接口让你实现）

- 另一个，是表单的处理上。
为了处理表单，Struts要求你用ActionForm类来处理输入的参数。你需要创建一个类将表单提交的数据映射到你的领域对象中。Spring允许直接将表单提交的参数绑定到对象中，能简化维护工作。

- Struts自身带有声明式表单验证支持，Spring中需要集成额外的验证框架。