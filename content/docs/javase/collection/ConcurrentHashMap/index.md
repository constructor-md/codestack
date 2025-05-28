---
title: ConcurrentHashMap
draft: false
weight: 2
---

# ConcurrentHashMap

是一个支持高并发更新和查询的哈希表，基于 HashMap 实现  
HashMap 本身没有对多线程情况进行线程安全的处理    
如果改用 HashTable 或者 Collections.synchronizedMap，则会对整个容器进行加锁，通知时间的其他操作都会被阻塞   
但是 ConcurrentHashMap 不对整个容器加锁，而是对一小部分加锁，不影响其他位置的操作  
## 数据结构
### JDK1.8 以前：Segement+ReentranLock（分段锁）

1. 由 HashMap 实现的 Segment 组成，实际是多段的 HashMap。可以理解为 ConcurrentHashMap 中有一个 Segement 数组，每个 Segement 就是一个 HashMap。
2. 采用分段锁的思想。写操作时，通过只锁涉及到的 Segment 的方式，保证修改的线程安全，同时不影响其它部分的并发操作。
3. Segment 内部进行扩容，和 HashMap 的扩容逻辑相似，先生成新数组，然后转移元素到新数组中。
4. 每个 Segment 单独扩容，互不干扰。是否需要扩容也是 Segment 内部单独判断，是否超过阈值
5. 默认创建 16 个 Segment，即默认最多允许并发 16 个不同的写
6. 并发度是Segment数量，比较固定

#### Segment 实现的弊端

1. 寻找元素需要两次 Hash
2. 提供了并发级别的默认设置，默认 16 个 Segment，无法根据实际场景动态调整，但是不一定适用于大部分场景，最好是让用户自己评估需要多少把锁
3. 存储成本比 HashMap 高，每个 Segment 最小容量为 2，默认并发级别 16，假如只需要存储 16 个元素，那么会建立一个 32 容量的 ConcurrentHashMap 来存

### JDK1.8 以后：Node+Synchronized+CAS 
1. 采用 Node 数组+链表/红黑树的数据结构（与HashMap高度相似 Node也只是一个数据节点）
2. 当链表长度超过 8 且数组长度 ≥ 64 时，链表转换为红黑树，提升查询效率
3. 每次操作某个槽位，都以该槽位的第一个元素作为锁，通过 synchronized 加锁，其他节点内仍可并发。锁粒度降到最低
4. 依赖 JVM 对 synchronized 的轻量级锁优化，性能优于显式 ReentrantLock
5. 槽位为空时（无锁写入），通过 CAS 原子操作直接写入数据
6. 移除 Segment，降低锁粒度、减少内存开销和哈希计算次数
7. 无锁读：通过 volatile 修饰的 Node 数组保证可见性，直接读取头节点。若为链表，遍历链表；若为红黑树，通过树结构查找
8. 并发度是理论上限为数组长度，随着数据变化动态调整

#### put 操作流程

1. 数组是否初始化，没有则初始化数组
2. 插入位置是否为空，是则使用 CAS 写入，size+1 并检查是否需要扩容
3. 是否正在扩容，是则协助扩容，扩容完毕后再从头判断起
4. 当前节点加锁，插入数据，解锁，szie+1 并检查是否需要扩容
