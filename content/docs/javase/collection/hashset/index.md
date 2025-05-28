---
title: HashSet
draft: false
weight: 5
---

# HashSet
## 数据结构
- 内部维护一个HashMap，new HashSet实际是new HashMap  
- 添加元素时，查找HashMap是否有元素，没有存放则直接添加，存放了则调用equals比较   
- 如果相同则放弃添加，不同则添加到最后   
## 注意点
- 只能存放一个Null值  
- 线程不安全
- 不能有重复元素