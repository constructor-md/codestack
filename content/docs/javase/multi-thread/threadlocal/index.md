---
title: ThreadLocal
draft: false
weight: 5
---

# ThreadLocal
- ThreadLocal是线程自己的私有变量副本，仅属于当前线程
- 每个线程Thread对象中都带有一个ThreadLocalMap对象，我们可以创建一个ThreadLocal作为该Map的Key，然后在Value中存储想要的值
- 每一个ThreadLocal对象都有独一无二的ThreadLocalHashCode值
  
- ThreadLocal set的时候，实际上是拿到Thread里面的ThreadLocalMap来进行Set
- ThreadLocal对象使用完毕后，应该进行回收，手动调用ThreadLocal的remove方法  
避免线程不被回收时（如线程池），ThreadLocalMap中的对象不被回收，从而出现内存泄漏

## 经典使用场景
在多个层级的方法中传递某个参数  
为了避免在方法参数列表上每个都加上这些参数带来的冗余  
写入ThreadLocalMap中存储  
如连接管理，某个连接对象要在不同的方法间传递，但同时线程之间又不共享连接  
（减少同一个线程内多个函数或者组件之间公共变量的传递的复杂度）  

## 可能出现内存泄露的原因
内存泄漏指的是程序中已经动态分配的堆内存由于某种原因未释放或无法释放，造成系统内存浪费，最终导致程序运行速度变慢等严重后果  
如果有一个对象，在将来再也不会被使用，但是就是没有被回收，那么认为该对象属于内存泄漏  


1. ThreadLocalMap的生命周期与Thread一样长，如果Thread执行完某个业务后，不删除对应ThreadLocal，且线程不被销毁（线程池线程），内部保持着的内容显然对系统是无意义的，这就是一种内存泄漏。
2. 一些ThreadLocal错误使用导致内存泄露的场景：
  - ThreadLocal定义为局部变量，线程执行完后没有remove，线程未被销毁。则ThreadLocal对象的强引用消失，仅剩ThreadLocalMap中的弱引用Key，GC仍然会把对象回收。此时ThreadLocalMap就有一个Entry的key为null，不会被别人访问，也不会被GC回收。出现内存泄露。
  - ThreadLocal定义为类变量，但是我们为了防止业务上的问题，在不理解ThreadLocal原理的情况下，每次都进入方法前去重新实例化ThreadLocal，业务结束后错误地将ThreadLocal设为null想以此避免业务上的问题，则以前的一些ThreadLocal对象强引用消失，出现上一个问题的内存泄漏。
  - ThreadLocal定义为类变量，一个线程执行完毕后没有remove，且线程未被销毁。后续该线程再也没有被用来执行该业务，以至于内部的ThreadLocal根本没有被访问的机会，这也是一种内存泄漏。
  - 在ThreadLocalMap中如果有Entry的key为null时，会在下一次调用set\get\remove方法时进行清理。但是如果后续我们再也没有访问过这个ThreadLocalMap，那自然出现上一个问题的内存泄漏。


## 为什么ThreadLocalMap要使用弱引用key？
- 如果key为强引用，如果我们用完不去手动删除，ThreadLocal对象由于强引用不会被回收，则Entry会内存泄露
- 如果Key为弱引用，我们不去手动删除，ThreadLocal没有其他强引用的情况下，GC回收掉ThreadLocal，ThreadLocalMap有机会在下一次访问清理无用的ThreadLocal，尽量避免内存泄露  
  
内存泄露是因为ThreadLocalMap和Thread生命周期一致，如果不去手动删除，ThreadLocal很可能是无意义的内容，这是一种内存泄漏  
弱引用是用来尽量减少内存泄露的一种举措，但不够完善，仍有内存泄露的可能，即后续没有访问哪些方法导致Entry不被清理，而不是内存泄漏的产生原因  
内存泄露的产生原因是我们的错误使用方法。  

## 最佳实践
ThreadLocal定义为静态不可变常量  
使用完对应的value后，调用ThreadLocal.remove清理  
