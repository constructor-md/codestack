---
title: CopyOnWriteArrayList
draft: false
weight: 3
---

# CopyOnWriteArrayList
## 数据结构
相当于一个线程安全的ArrayList，内部存储仍使用数组实现  
```
public class CopyOnWriteArrayList<E> implements List<E> {
    private transient volatile Object[] array;  // 使用volatile保证可见性
    final transient ReentrantLock lock = new ReentrantLock();  // 写锁
}
```

## 线程安全机制
线程安全通过ReentrantLock实现，允许多线程并发读取，但只允许一个线程写入  
1. 内部维护final transient 的ReentrantLock对象作为全局锁
- transient阻止默认序列化
- volatile保证可见性和有序性
2. 写操作首先进行加锁，结束后解锁。保证并发写的线程安全。
3. 添加新元素时，复制一个新数组，在新数组上添加。而此时并发的读操作则在原数组上进行。通过复制元素读写分离，来保证读写并发的性能。
4. 读到的可能不是最新的数据，不适合实时性要求很高的场景。
5. 采用写时复制思想（Copy on Write），适合读多写少的并发场景

## 局限性
- 内存占用高：每次写操作都创建新数组，可能导致频繁 GC。  
  （需要注意大数组情况性能很差）
- 写操作性能差：涉及数组复制，写操作时间复杂度为 O (n)。  
  （若需要强一致性且写操作较多，使用 Collections.synchronizedList）
- 弱一致性：读操作可能看不到最新数据，不适合实时性要求高的场景  