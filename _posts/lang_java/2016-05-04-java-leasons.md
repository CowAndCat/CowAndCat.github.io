---
layout: post
title: Java编码教训
category: java
comments: false
---

## 多进程编程

在一个进程里面开了新建了两个线程，发现有的时候只启动了其中的一个，有的时候两个都能够正常运行，
这个时候，应该是进程创建的问题，最快且有效的方法是在两个创建动作之间加入延迟。

例如：


    new ScheduledThreadPoolExecutor(10).execute(new MonitorForumPostServiceJobTaskThread());  
    Thread.sleep(1000); //延迟启动1s  
    new ScheduledThreadPoolExecutor(10).execute(new MonitorForumPostServiceJobTaskReview());  
