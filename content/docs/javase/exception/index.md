---
title: 异常体系
draft: false
weight: 10
---

# 异常体系

## Throwable

Throwable 类是所有异常和错误的超类  
两个直接子类为 Error 和 Exception，分别表示错误和异常

## Error

Error 指的是程序无法处理的错误，由 JVM 产生和抛出

- 如 OutOfMemoryError、StackOverFlowError、ThreadDeath 等。Error 发生时，JVM 会选择终止线程

## Exception

分为不检查异常（unchecked Exception）和检查异常（checked exception）  
也是运行时异常（RuntimeException）和非运行时异常

- Exception 是程序可以处理的异常，分为两大类，运行时异常和非运行时异常。程序中需要尽量去解决这些异常

### 运行时异常

指 RuntimeException 类及其子类，如 NullPointerException、IndexOutOfBoundsException 等。

- 属于不检查异常，程序可以选择捕获处理，也可以不处理
  例如：
- NullPointerException - 空指针引用异常
- ClassCastException - 类型强制转换异常
- IllegalArgumentException - 传递非法参数异常
- ArithmeticException - 算术运算异常
- ArrayStoreException - 向数组中存放与声明类型不兼容对象异常
- IndexOutOfBoundsException - 下标越界异常
- NegativeArraySizeException - 创建一个大小为负数的数组错误异常
- NumberFormatException - 数字格式异常
- SecurityException - 安全异常
- UnsupportedOperationException - 不支持的操作异常

### 非运行时异常

指 Exception 类及其子类中 RuntimeException 类及其子类以外的类

- 如 IOException、SQLException 等，以及用户自定义的 Exception（一般不会自定义检查异常，而是自定义 RuntimeException）
- 属于检查异常，如果程序不去捕获并处理或抛出到最外层，则编译不能通过
