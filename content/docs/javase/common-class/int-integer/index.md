---
title: int|Integer
draft: false
weight: 5
---

# int和Integer的区别

1. 数据类型不同，int是基本数据类型，Integer是引用数据类型
2. 默认值不同，int默认值为0，Integer默认值为null
3. 存储方式不同
  1. int在内存中直接存储数值，如果是成员变量则放在堆内的对象中，如果是局部变量则放在线程私有的虚拟机栈中
  2. Integer是引用类型，存储的是一个对象地址，new时是生成一个指针指向该对象，实际对象在堆中
4. 实例化方式不同，Integer必须实例化才能使用，int不需要
5. 变量比较方式不同，int之间可以==比较，Integer之间最好equals比较

int和Integer之间的比较：  
- int之间可以==比较
- int和Integer之间比较时，Integer自从拆箱变为int进行比较，可以==比较
- Integer和Integer之间比较，由于内部缓存了值为-128-127的对象，对这个范围内可以==比较，对其他范围必须eaquls比较，equals比较时是在比较内部的int值

