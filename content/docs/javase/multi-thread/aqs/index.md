---
title: AQS
draft: false
weight: 6
---

# AQS
AQS：AbstractQueuedSynchronized类  
JUC包中的ReentrantLock、Semaphroe、ReentrantReadWriteLock、CountDownLatch等几乎所有类都依赖于AQS实现  

## 重要属性
AQS有两个重要属性：
1. 以Node为节点实现的双向链表、先进先出队列，节点内包装线程
2. STATE标志，通过CAS改变其值，象征锁资源

## 部分逻辑
线程抢占资源，就是尝试设置STATE标志位的值，ReentrantLock就是将其设置为1，如果重入则不断自增  
如果STATE > 0则该资源被占用，即锁定，其他线程不能获得锁  
    
线程抢占资源失败，则进入等锁队列中，等待头节点释放锁后唤醒后继节点得锁（公平锁），或唤醒所有节点抢锁（非公平锁）  
  
AQS是一个抽象类，继承AQS自己实现的同步器，只需要根据同步器需要满足得性质取实现线程获取和修改同步状态变量的方式，队列的维护已经被顶层实现好。  
仅需要实现父类的一些抽象方法，方法内部设置同步状态变量即可  

AQS独占锁：  
ReentrantLock、ReentrantWriteLock等  
AQS共享锁：  
ReentrantReadLock、Semaphore、ConutDownLatch等  