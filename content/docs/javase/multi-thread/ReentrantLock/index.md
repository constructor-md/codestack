---
title: ReentrantLock
draft: false
weight: 8
---

# ReentrantLock
## 介绍
JDK5时，没有synchronized优化，该关键字加锁是个比较重的操作  
J.U.C.locks.Lock接口提供了基于类的全新互斥手段，后续扩展出了不同调度算法、不同特征、不同性能不同于一的各种锁  

重入锁（ReentrantLock）是Lock的最常见实现，是可重入的  
基本用法和synchronized相似，相比synchronized主要增加了三个高级功能：  
1. 等待可中断  
持锁线程长期不释放锁，正在等待的线程可以放弃等待，改为处理其他事情
2. 公平锁  
多个线程等待同个锁时，必须按照申请所得时间顺序来依次获得锁  
非公平锁在锁什邡市，任何等锁线程都能有机会获得锁  
synchronized是非公平的，ReentrantLock默认也是非公平，但是可以通过布尔参数改为公平锁  
使用公平锁后ReentrantLock性能急剧下降，影响吞吐量  
3. 锁绑定多个条件  
一个ReentrantLock可以绑定多个Condition对象  
synchronized中，锁对象的wait、notify或notifyAll方法配合可以实现一个隐含条件，但是如果要和多于一个的条件关联，则必须额外添加锁  
但ReentrantLock则多次调用newCondition即可  
  
## 锁释放
使用Lock需要保证在Finally块中释放锁，否则抛出异常后可能永远不会释放锁  
synchronized则由虚拟机保证锁的正常释放  


## 基本实现原理
Lock接口的实现内部封装了一个AQS同步队列  
AQS同步队列支持先进先出，数据结构为双向链表，维护头尾指针，方便从任意一个位置访问前后节点  
线程如果得锁失败，就会被封装成Node进入等锁队列  
当获取锁的线程释放锁，就从队列中取出一个节点唤醒线程    
  
一个线程得锁失败，则加入队列尾，通过尾结点指针对接队列尾  
通过CAS将tail指向新的尾节点  
一个线程释放锁，唤醒后继节点，后继节点得锁成功后，会把自己设为队列头节点  
设置头节点不需要用CAS，因为设置头节点是获取锁的线程来完成  
同步锁只能由一个线程获取，所以不需要CAS保证  
  
  
ReentraintLock锁对象，通过CAS进行加锁，如果加锁失败，则走锁竞争的逻辑  
加锁成功则走加锁成功逻辑  
  
加锁成功后，会设置state属性值，支持可重入，同个每次加锁state值加一，解锁则减一，如果state值大一0，说明此时有线程持有锁，=0则是五锁状态  

## 公平/非公平锁
公平锁就是锁释放后唤醒后继节点获得锁    
非公平锁就是唤醒全部线程，让他们都CAS一下，谁成功谁就得锁，不成功就进入队列等待下一次  