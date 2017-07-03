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

## 2. 优化代码
    
    long t = System.currentTimeMillis();
    Long sum = 0L;
    for (long i = 0; i < Integer.MAX_VALUE; i++) {
        sum += i;
    }
    System.out.println("total:" + sum);
    System.out.println("processing time: " + (System.currentTimeMillis() - t) + " ms");

优化1略.（9803ms->1343ms）

优化2：（->750ms）

        long t = System.currentTimeMillis();
        FutureTask<Long> future = new FutureTask<Long>(new Callable<Long>() {
            @Override
            public Long call() throws Exception {
                long sum = 0;
                for (long i = 0; i < Integer.MAX_VALUE/2; i++) {
                    sum += i;
                }
                return sum;
            }
        });

        new Thread(future).start();
        
        long sum = 0L;
        for (long i = Integer.MAX_VALUE/2; i < Integer.MAX_VALUE; i++) {
            sum += i;
        }
        
        sum += future.get();
        
        System.out.println("total:" + sum);
        System.out.println("processing time: " + (System.currentTimeMillis() - t) + " ms");