---
title: 迭代器
draft: false
weight: 6
---

# 迭代器
集合类的顶层接口Collection继承了Iterable接口  
Iterable接口有一个方法   
```
# 返回一个Iterator对象
Iterator<T> iterator();
``` 
Iterator接口，有三个方法：hashNext、next、remove  
各个集合中实现了Iterator内部类，实现了这几个方法  
  
使用时，首先获取集合对象中的Iterator对象，然后while循环判断hashNext，然后用Next获取数据  
迭代过程中可以使用迭代器的remove方法删除元素，不能使用集合的remove方法删除元素  
原因是迭代器会更新modCount和expectedModCount，使得判断可以通过，不用抛出异常  
这在单线程下没问题，在多线程下不安全  

增强for循环：   
底层使用Iterator实现，只能遍历集合或数组  