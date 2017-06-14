---
layout: post
title: 面试Java程序员的题
category: java
comments: false
---
## 1. 填写输出，解释原因
public static void main(String[] args) {
    double num = 1.23;
    System.out.println((int) num); // 1
    System.out.println(true ? (int) num : num); // 1.0
}
