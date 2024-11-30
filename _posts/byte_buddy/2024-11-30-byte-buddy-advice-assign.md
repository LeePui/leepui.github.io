---
layout:     post
title:      "获取覆盖参数及类变量(@Advice.Assignxxx相关注解说明)"
subtitle:   "ByteBuddy @Advice.Assign 注解使用详解"
date:       2024-11-30 10:50:46
author:     "国道蛋黄派"
catalog: true
published: true
header-style: text
categories: 
  - byte-buddy
tags:
  - byte-buddy
---

# @Advice.Assignxxx 能实现的功能有：
- 获取方法的参数
- 覆盖方法的参数
- 获取方法的返回值
- 覆盖方法的返回值
- 获取类的变量
- 覆盖类的变量


# 注解说明
- 在覆盖参数时，会使用到这个注解：
  - `@Advice.AssignReturned.ToArguments(@ToArgument(index = 2, value = 1, typing = DYNAMIC))`
  - ToArguments说明可以覆盖多个参数，每一个覆盖的参数使用ToArgument注解来声明。
- 这里主要说明index，valuse两个变量的作用
  - index代表增强方法返回的Object数组中第几个元素，下标从0开始。
  - value代表被增强的方法中的第几个参数，下标从0开始。
  - 上面的注解的意思就是：用Object数组的第三个元素覆盖被增强方法的第二个参数。

# 举例
### 被增强的接口如下：
![增强的接口](/img/in-post/byte-buddy/byte-buddy-advice-assign/1.png)
请求如下：`localhost:8080/test?a=a&b=true&c=10&d=d`  
正常打印：`接口收到参数, a: a, b: true, c: 10, d: d`  
此时我想将第四个参数，也就是d的值改一下，改成：DDD  

### 增强的代码：
![增强的代码](/img\in-post\byte-buddy\byte-buddy-advice-assign\2.jpg)
请求如下：`localhost:8080/test?a=a&b=true&c=10&d=d`  
  
控制台打印结果：
![控制台打印结果](/img\in-post\byte-buddy\byte-buddy-advice-assign\3.jpg)  
  
反编译结果：（反编译的结果不会太准确，但也能看出来赋值的过程。）
![反编译结果](/img\in-post\byte-buddy\byte-buddy-advice-assign\4.jpg)
>根据反编译和注解@ToArgument(value = 3, index = 0, typing = Assigner.Typing.DYNAMIC)再次说明结论：增强的方法可以指定返回一个Object[]数组，这个数组的第一位(index=0)，将被赋值给string2(d)，d是第四个参数(value=3)。
  
  
---
### 如何获取方法的参数、覆盖方法参数、获取返回值、覆盖返回值
![如何获取方法的参数、覆盖方法参数、获取返回值、覆盖返回值](/img\in-post\byte-buddy\byte-buddy-advice-assign\5.jpg)

```java
public class DemoControllerTestAdvice {
 
    /**
     * 方法进入
     *  1、替换单个参数，方法不需要使用Object数组，直接使用该对象即可，然后注解这样写:@Advice.AssignReturned.ToArguments(@ToArgument(value = 位置, typing = Assigner.Typing.DYNAMIC))
     */
    @Advice.OnMethodEnter(suppress = Throwable.class, inline = false)
    @Advice.AssignReturned.ToArguments({@ToArgument(value = 3, index = 0, typing = Assigner.Typing.DYNAMIC),})
    public static Object[] beforebefore(@Advice.This Object obj,
                              @Advice.Argument(0) String a, @Advice.Argument(1) Boolean bool, @Advice.Argument(2) Integer integer, @Advice.Argument(3) String d) {
        List<String> arrs = new ArrayList<>();
        arrs.add("1a");
        arrs.add("2b");
        arrs.add("3c");
        arrs.add("4d");
        System.out.println();
        System.out.println("参数0：" + a);
        System.out.println("参数1：" + bool);
        System.out.println("参数2：" + integer);
        System.out.println("参数3：" + d);
        return new Object[]{"DDD", arrs};
    }
 
    @Advice.OnMethodExit(suppress = Throwable.class, inline = false)
    @Advice.AssignReturned.ToReturned
    public static String afterafter(@Advice.Return String result, @Advice.Enter Object[] params) {
        System.out.println("接口原始返回结果: " + result);
        String a = result + "我是修改后的";
        System.out.println("修改后的接口：" + a);
 
        if (params != null) {
            List<String> arrs = (List<String>)params[1];
            System.out.println("自己存的值: " + arrs.stream().collect(Collectors.joining(",")));
        }
        return a;
    }
}
```
  
---
### 获取类的变量、覆盖类变量
![获取类的变量、覆盖类变量](/img\in-post\byte-buddy\byte-buddy-advice-assign\6.jpg)
