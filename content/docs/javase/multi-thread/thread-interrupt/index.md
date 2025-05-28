---
title: 线程中断
draft: false
weight: 3
---

# 线程中断
## 如何中断线程？
调用线程对象的interrupt方法中断指定线程

## 如何判断当前线程是否被中断？
调用线程对象的isInterrupted方法


## 线程中断线程就停止了吗？
程序中，我们不能随便中断一个线程，因为十分不安全，因为我们不知道线程当前处于何种状态  
强行中断可能会导致锁不能释放、操作数据库可能导致事务无法提交等  
Java中将线程的Stop方法设置为过时，禁止使用。  
当我们执行线程的interrupt方法时，本质不是中断线程，而是设置一个标志位  
当一个线程的interrupt方法被调用：  
1. 如果线程当前阻塞，如io、wait，会立马退出阻塞，并抛出Interruption异常，我们通过捕获Interruption异常，做一定处理，然后让线程结束方法退出  
2. 如果线程在运行中，则不受影响继续运行。仅仅是线程的中断标记被设置为true，需要线程自己在适当的位置检查isInterrupted方法来查看自己是否被中断，并作退出操作  
3. 值得一提的是，Thread.isInterrupted() 判断中断异常后，中断标志位会变为false，相当于中断标志位被清空。如果后面还想判断执行停止，此时需要手动再次中断或者设置标志位  
```
# Java实现 Thread类
public static boolean interrupted() {
    return currentThread().isInterrupted(true);
}

public boolean isInterrupted() {
    return isInterrupted(false);
}

private native boolean isInterrupted(boolean ClearInterrupted);
```

## 如果一个任务创建了线程池来执行子任务，需要中断主任务和子任务
则在主任务的中断操作中，调用线程池的中断方法停止线程池
1. shutdown方法  
拒接新任务，新任务提交时执行拒绝策略。  
但是队列中的任务仍会继续执行直到完毕，中断空闲线程
2. shutdownNow方法  
拒接新任务，新任务提交执行拒绝策略。    
调用所有线程的interrupt方法进行终端，子任务需要自己执行结束任务。  
任务队列中的任务不会继续执行，并返回被抛弃的任务List  








