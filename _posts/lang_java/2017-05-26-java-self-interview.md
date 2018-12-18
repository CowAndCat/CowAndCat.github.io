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

如果?后的表达式1和表达式2的类型不相同，那么会参考它们的交集类型进行类型转换。

确定条件表达式结果类型的规则的核心是以下3点：

1. 如果表达式1和表达式2操作数具有相同的类型，那么它就是条件表达式的类型。
2. 如果一个表达式的类型是byte、short、char类型的，而另外一个是int类型的常量表达式，且它的值可以用类型byte、short、char三者之一表示的，那么条件表达式的类型就是三者之一
3. 否则，将对操作数类型进行二进制数字提升，而条件表达式的类型就是第二个和第三个操作数被提升之后的类型。

所以上面输出的1.0实际上是对返回结果的类型提升到了double。

## 2. 优化代码
    
    long t = System.currentTimeMillis();
    Long sum = 0L;
    for (long i = 0; i < Integer.MAX_VALUE; i++) {
        sum += i;
    }
    System.out.println("total:" + sum);
    System.out.println("processing time: " + (System.currentTimeMillis() - t) + " ms");

优化1.（9803ms->1343ms）

将Long sum换成 long sum, 减少自动封装带来的性能损耗。

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