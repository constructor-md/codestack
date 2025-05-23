---
title: InnoDB 事务
draft: false
weight: 2
---


# InnoDB 事务
MySQL中，事务功能主要是由 InnoDB 存储引擎来实现的
## InnoDB 事务特性（ACID）及实现
### 原子性（Atomicity）
事务内的操作要么全部成功，要么全部失败  

通过Undo Log实现原子性  
Undo Log 记录事务操作的反操作，或者说记录了每个操作前的数据，当事务需要回滚时，可以根据Undo Log恢复数据。  
从而实现事务内的操作要么全部执行，要么都不执行

### 一致性（Consistency）
事务必须使数据库从一个一致性状态变换到另一个一致性状态  
依赖于原子性和隔离性的实现，还依赖于数据库自身的完整性约束(如外键、CHECK 约束等)和应用程序的正确逻辑


### 持久性（Durability）
一个事务一旦被提交了，那么对数据库中的数据的改变就是永久性的，即便是在数据库系统遇到故障的情况下也不会丢失提交事务的操作  
通过数据写入磁盘，以及Redo Log记录写入内存未写入磁盘的操作，在系统崩溃恢复时写入磁盘。保证持久性  
Redo Log本身在磁盘顺序写入，速度很快

### 隔离性（Isolation）
多个并发事务之间要相互隔离。对于任意两个并发的事务T1和T2，在事务T1看来，T2要么在T1开始之前就已经结束，要么在T1结束之后才开始

通过MVCC和锁机制实现隔离性

先说明MVCC隔离性实现原理，锁机制放在锁的篇章中

### MVCC
innodb通过ReadView快照和版本链实现MVCC  
#### ReadView 快照
Readview是代码中的对象，主要有如下属性：
- m_ids: 生成ReadView时当前系统中活跃的读写事务的事务id列表（未提交的事务）
- min_trx_id: 生成ReadView时当前系统中活跃的读写事务的最小事务id m_ids的最小值（最早的未提交事务）
- max_trx_id: 生成ReadView时系统应该分配给下一个事务的id值（下一个事务）
- creator_trx_id: 生成该ReadView的事务的id（所属事务）

#### 版本链
在Innodb中，以最新记录和 undo log 中的历史记录形成了版本链  
Innodb的每行数据，都有两个隐藏字段：事务ID和回滚指针  
- 事务ID代表这行数据由哪个事务最后更新，回滚指针指向Undo Log中这行数据的上一个版本  
- Undo Log中记录了数据的每个版本，同样也携带着修改成该版本的事务ID和指向上一个版本的回滚指针  
- 每行最新的数据，以及Undo Log中的历史版本中的数据，的回滚指针们形成了一条链表，即版本链  
- 事务的每个修改，不论是否已经提交，都会被记录在版本链上

> MySQL 中有专门的 Purge 线程，会定期检查 Undo Log，删除那些已经不再需要的记录  
> Purge 操作会遍历 Undo Log 链表，找到那些没有被任何活动事务引用的节点，并将其从磁盘上删除，释放空间

#### 以可重复读为例说明MVCC的实现
在事务隔离级别为可重复读情况下，执行select前，会先生成ReadView  
通过ReadView在版本链中进行比较，可知版本链上的哪些数据是可以被自己访问的  
trx_id是该行的版本链上某个版本的事务ID
- trx_id == creator_trx_id Readview自己修改的数据，可以访问
- trx_id < min_trx_id Readview创建之前的已提交数据，可以访问
- trx_id > max_trx_id 创建在Readview事务之后的数据，不可访问
- min_trx_id <= trx_id <= max_trx_id Readview创建时尚未提交的数据，不可访问

所以：通过ReadView的事务信息，以及版本链上数据的事务信息，以及事务ID的全局自增，可以得知当前事务可以访问哪些数据！

对可重复读而言，仅在第一次select时生成ReadView即可  
此后每次查询，访问到的数据都是以该ReadView为基准  
从而每次要么访问到自己的最新修改，要么一直是生成当时的最新提交数据，不会变化  
同理可看其他事务级别：
- 读已提交：
每次select都会生成ReadView，新的ReadView可以访问新提交的数据  
于是每次select都可以访问到最新的提交数据，或者是自己的修改数据
- 读未提交：
每次直接访问最新的版本链数据，不关心是否提交，不需要MVCC
- 串行化：
通过加锁互斥访问，不需要MVCC




































# MVCC 多版本并发控制


## 版本链
数据库中，以最新记录和undolog中的历史记录形成了版本链

版本链中每个版本，除了具体行数据之外，还有该版本的事务ID、和回滚指针，每一行记录都有这两个隐藏列

每个版本的回滚指针指向上一个版本

## ReadView 快照数据
作用：让数据库知道在具体查询的时候，应该去查询哪个版本的数据

数据结构：代码里的一个对象
- m_ids: 生成ReadView时当前系统中活跃的读写事务的事务id列表（未提交）
- min_trax_id: 生成ReadView时当前系统中活跃的读写事务的最小事务id m_ids的最小值（最老的一个未提交）
- max_trx_id: 生成ReadView时系统应该分配给下一个事务的id值（未生成即将生成）
- creator_trx_id: 生成该ReadView的事务的id（创建者）

ReadView如何判断版本链中哪个版本可用？
- Trx_id == creator_trx_id 可以访问这个版本（创建者可以访问）
- Trx_id < min_trx_id 可以访问这个版本（可以访问已提交的数据）
- Trx_id > max_trx_id 不可以访问这个版本（当前事务不能访问超过当前版本链的数据）
- Min_trx_id <= trx_id <= max_trx_id 如果trx_id在m_ids中，就可以不访问这个版本，因为里面的数据都没有commit

## ReadView如何实现读已提交和可重复读？
版本链是全局的，没有事务时版本链的头部是最新数据，后面是每个版本的历史数据

如果有多个事务同时更新某一行数据，每个事务的更新，都会在版本链上生成一个版本

1. 执行一个select前
先生成一个当前事务的ReadView，ReadView包含上述数据
2. 让ReadView在版本链从头到尾比较：(版本数据对查询的可见性)
- 如果该版本是当前事务提交的，那自然可以访问该数据，且该版本未提交也可以访问
- 如果该版本不是当前事务提交，但是是已提交的数据，当然可以访问
- 如果该版本数据是当前事务之后生成的事务生成的版本，那么肯定没有提交，不能访问
- 如果该版本数据是当前事务生成ReadView时的未提交事务之一，那么不能访问这些未提交的数据
## 事务隔离级别与MVCC
- 如果是读已提交：
    则每次select都会生成ReadView，使得除了creator_trx_id以外的数据有所变化

    于是每次select都可以访问到最新的提交数据，或者是自己的修改数据

    访问到的最新提交数据可能变化
    
- 如果是可重复读：
    仅在第一次select生成ReadView

    从而每次要么访问到自己的最新修改，要么一直是生成当时的最新提交数据，不会变化

- 如果是读未提交：
    则每次直接访问最新的版本链数据，不关心是否提交，不需要MVCC
    
- 如果是串行化：
    则通过加锁互斥访问，不需要MVCC
    

> 幻读问题的解决：
>
>    可明确的是：仅在快照读的情况下可以解决幻读，当前读不能解决幻读
>
>   幻读指的是两次查询前后，因为中间有事务提交，导致查询得到的数据量不符合









