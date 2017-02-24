---
layout: post
title: Web面试题集2
category: web
comments: false
---
## 1、 同源策略
同源策略（Same origin policy）是一种约定，它是浏览器最核心也最基本的安全功能，如果缺少了同源策略，则浏览器的正常功能可能都会受到影响。可以说Web是构建在同源策略基础之上的，浏览器只是针对同源策略的一种实现。

所谓同源是指，域名，协议，端口相同。

当一个浏览器的两个tab页中分别打开百度和谷歌的页面，当浏览器的百度tab页执行一个脚本的时候会检查这个脚本是属于哪个页面的，即检查是否同源，只有和百度同源的脚本才会被执行。

#### 1.1 同源策略： 
浏览器的同源策略，限制了来自不同源的"document"或脚本，对当前"document"读取或设置某些属性。**从一个域上加载的脚本不允许访问另外一个域的文档属性。**

 在浏览器中，`<script>、< img>、<iframe>、<link>`等标签都可以加载跨域资源，而不受同源限制，但浏览器限制了JavaScript的权限使其不能读、写加载的内容。

#### 1.2 跨域实现：Ajax跨域技术

- **JSONP**:就是利用`<script>`标签的跨域能力实现跨域数据的访问，请求动态生成的JavaScript脚本同时带一个callback函数名作为参数。其中callback函数本地文档的JavaScript函数，服务器端动态生成的脚本会产生数据，并在代码中以产生的数据为参数调用callback函数。当这段脚本加载到本地文档时，callback函数就被调用。
- **Proxy**: 使用代理方式跨域更加直接，因为SOP的限制是浏览器实现的。如果请求不是从浏览器发起的，就不存在跨域问题了。
- **CORS**(Cross origin resource sharing)

其他：Flash/SilverLight跨域。

#### 1.3 Cookie 同源策略
Cookie中的同源只关注域名，忽略协议和端口。所以https://localhost:8080/和http://localhost:8081/的Cookie是共享的。