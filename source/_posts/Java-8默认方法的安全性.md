---
title: Java 8默认方法的安全性
date: 2019-03-15 00:22:20
categories:
- java
tags:
- java8
- 默认方法
- 写给大忙人看的Java 8 SE
- default method
---

# 背景

《写给大忙人看的Java 8 SE》第一章最后一个习题问：
> 在过去，你知道向接口中添加方法是一种不好的形式，因为它会破坏已有的代码。现在你知道了可以向接口添加新方法，同时能够提供一个默认的实现。这样做的安全程度如何？描述一个Collection接口的新stream方法会导致遗留代码编译失败的场景。二进制的兼容性如何？JAR文件中的遗留代码是否还能运行？`

给接口添加默认方法还会导致老代码编译失败？😲 脑海里的第一个想法是，这TM在开玩笑吧？至少JDK容器代码已经这么做了呀，Java添加新功能兼容性肯定是放在第一位，比如泛型。再细读两遍题干，确实是陈述句，还特别点名了stream方法，说明确有其事。

我不禁陷入了沉思。。😔😔😔


# 解答

题目有三问，以下逐个解答：

## stream方法会导致编译失败的场景

### 默认方法的限制

如果一个类的父类、父接口含有签名相同的方法，且至少有一个是默认方法，此时需要解决冲突：

1. 如果一个接口提供了默认方法，父类中也有一个实现，则选择父类中的方法。

2. 类的父接口们存在相同的方法签名（且至少有一个默认方法），则必须通过覆盖（提供实现）来解决冲突
    * 可以在实现中调用指定父接口的默认方法 XXX.super.getName()
    * 只要其中一个接口提供了默认实现就需要手动解决冲突

总结起来就是：类优先原则，类中的方法总是优先于默认方法。因此**不可能**为Object中的方法定义一个默认方法。

### 解题思路

> stream方法是`java.util.Collection`中的默认方法

从方法签名冲突的思路入手，找到一种编译错误的场景：

#### 同名方法返回值不同

因为Stream类当时还不存在，返回值肯定不会是新增加的`java.util.stream.Stream`

```java
public class Ex12 {

  public interface OldStream {

    int stream();
  }

  public static class OldColl extends ArrayList<Integer> implements OldStream {

    @Override
    public int stream() {
      return 1;
    }
  }
}
```

> 'stream()' in 'java.util.Collection' clashes with 'stream()' in 'xyz.quxiao.java8forimpatient.chap1.Ex12.OldStream'; attempting to use incompatible return type

// TODO 这是定义类方法的限制

## 二进制的兼容性

## JAR中的遗留代码是否还能运行