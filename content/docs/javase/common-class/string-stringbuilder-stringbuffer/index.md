---
title: String|StringBuilder|StringBuffer
draft: false
weight: 2
---

# String|StringBuilder|StringBuffer
## String 
- String是不可变字符串对象  
- 如果进行修改，会生成一个新的字符串赋给引用  
- 由于不可变，所以线程安全
- String的字面量值会被放入字符串常量池
- 当进行字符串的equals比较，由于String重写了equals方法，调用时是比较每一个char字符来判断是否相等
- 当进行==比较时，比较的是对象的内存地址是否相等
- 如果使用字面量“”== String对象，则实际上比较的是字符串常量池中该字面量的地址和堆中String的地址，结果肯定是不相等
- 当使用==比较两个String，结果也是不相等
- 可以使用String.intern方法取出字符串常量池的地址，从而==判断可以相等
- 当需要比较字符串是否相等时，equals永远是不会出错的方法
- String a = new String("xx");时，可能出现一个或两个对象，一个在堆里，字面量在常量池

## StringBuffer
线程安全，append方法加入了synchronized同步锁。效率低

## StringBuilder
线程不安全，append方法未加锁，但效率高
