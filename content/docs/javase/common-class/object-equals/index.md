---
title: 比较对象相等
draft: false
weight: 3
---

# 比较对象相等
对于8种基本数据类型，==比较的是操作数的值之间的关系  
其余一切皆是对象，==比较的是内存地址  
由Object继承来的equals方法比较的是内存地址  
  
Object规范约定：  
- 如果两个对象通过equals方法比较是相等的，则hashcode方法结果值必须相等
- 如果两个对象通过equals方法比较不相等，则不要求hashcode方法结果相等
当一个程序执行过程中，equals方法没有修改任何信息，同一个对象上重复调用hashcode方法，必须返回相同值  
两个应用互相调用，则hascode可以不一致。  
  
这意味着，当我们要重写equals方法时，必须重写hashcode方法。  
否则可能违反：同一对象的hashcode值必须相等的约定  
会使得一些使用hashcode的类使用出现问题  
如HashMap，存入时会调用类自身的hashcode方法得到hashcode；  