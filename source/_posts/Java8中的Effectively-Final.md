---
title: Java8中的Effectively Final
date: 2019-03-08 23:36:56
tags:
---

Java 8中引入了effectively final的概念，主要用在同样是这一版Java引入的lambda表达式中。意思是一个变量虽然是非final的，但自其定义后实际上不会发生改变。

Java 8之前的版本就有匿名类，没法在匿名类中引用任何非final的变量。lambda表达式是匿名类的语法糖，也就继承了这一特性，但只要变量生命周期中不会改变也能满足“final”的要求，所以Java 8放松了final的要求，只要实际final就可以了。

《写给大忙人看的Java 8 SE》第一章的习题8就是这个问题。它问:

```java
    String[] names = {"by", "gg", "bd"};
    for (String name : names) {
      new Thread(() -> System.out.println(name)).start();
    }
```

这里的线程运行的结果是否达到预期？即每个线程是否打印对应的名字，答案是正确，能实现要求。这里的name就是effectively final的，因为在它的生命周期（foreach循环中）是不变的。
而以下的代码是否正常？

```java
    for (int i = 0; i<names.length;i++) {
      new Thread(() -> System.out.println(names[i])).start();
    }
```

回答是不正常，它甚至报编译错误：
> Variable used in lambda expression should be final or effectively final

就是因为变量i在其生命周期中是变化的。lambda不能使用这样的变量。

怎么改？可以这样:

```java
    for (int i = 0; i < names.length;i++) {
      int index = i;
      new Thread(() -> System.out.println(names[index])).start();
    }
```

循环中声明一个effectively final的变量，lambda就能捕获到了。

如果你对第一个例子的effectively final仍有疑问，参考这个[文档](http://www.importnew.com/20198.html)