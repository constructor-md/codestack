---
title: Java中线程的实现
draft: false
weight: 4
---

# Java中线程的实现

## 线程与进程
线程是比进程更加轻量级的调度执行单位，引入线程可以将进程的资源分配和执行调度分开，各个线程共享进程的资源，又可以独立调度  
线程是Java中处理器资源调度的最基本单位  

## Java线程的实现
Thread对象内的许多关键方法都被声明为native  
native方法意味着这个方法没有使用或无法使用平台无关的手段来实现，即，Java线程的实现依赖于平台对线程的实现
以HotSpot为例，每一个Java线程都是直接映射到一个操作系统原生线程来实现，中间没有额外的间接结构  
HotSpot自身不会干涉线程调度，只能设置优先级给操作系统提供调度建议，全权交给操作系统处理  
什么时候冻结/唤醒/停止线程、该给线程分配多少处理器执行时间、线程分配给哪个处理器执行，都是操作系统完成，操作系统全权决定

## Java线程调度：
线程调度指Java为线程分配处理器使用权的过程。 
两种调度方式：协同式调度、抢占式调度  
Java使用的线程调度方式为抢占式调度  
1. 每个线程都是由系统分配执行时间，线程的切换线程自己不能决定。不会出现一个线程导致整个进程或系统阻塞的问题。
2. Thread::yield方法主动让出执行时间，但是不能主动获取执行时间
3. Java只能建议操作系统给一些线程分配多一点执行时间，另一些少一点。通过设置优先级来完成
4. Java一共设置了十个级别的优先级，两个线程同时处于Ready状态时，优先级高的容易被同选择执行
5. 操作系统仍是全权掌握决定权，甚至可以更改优先级。我们无法在程序中准确判断一组状态都为Ready的线程哪一个会先执行。

## Java线程状态转换
Java语言定义了线程的六种状态：  
1. 新建  
创建后尚未start的线程
2. 运行  
线程有可能在执行，有可能在等待操作系统分配执行时间
3. 无限期等待  
该状态的线程不会被分配处理器执行时间，必须等待其他线程显式唤醒  
如Thread.Join()，object.wait()的不设置Timeout的方法  
唤醒方法如notify()  
4. 限期等待  
等待期间不会被分配处理器调度时间，但是不需要其他线程显式唤醒，一定时间后系统会自动唤醒  
如sleep，有参的wait、join等方法  
5. 阻塞  
阻塞状态是在等待获取一个排他锁，将在另一个线程放弃一个排他锁时发生  
等待则是等待唤醒动作的出现  
6. 结束  
已终止的线程状态，线程结束执行

## wait和notify/notifyAll
- wait和notify/notifyAll方法是Object对象的实例方法 
```
// 对象锁
private static final Object lock = new Object();
private static int count = 0;

// 生产者
public static void producer() {
    synchronized (lock) {
        count++;
        System.out.println("生产数据，count=" + count);
        lock.notify(); // 唤醒消费者
    }
}

// 消费者
public static void consumer() {
    synchronized (lock) {
        while (count == 0) {
            try {
                System.out.println("等待数据...");
                lock.wait(); // 释放锁并等待
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        count--;
        System.out.println("消费数据，count=" + count);
    }
}
```
- wait方法必须在sychronized同步代码块下使用，否则会报错
- wait() 让线程释放锁并进入等待状态，需通过 同一对象的 notify()/notifyAll() 唤醒。
- notify() 只能唤醒在该对象锁上等待的线程，且唤醒后线程需重新竞争锁
- wait()和notify()/notifyAll()必须成对出现
没有 notify()，wait() 线程会一直阻塞；  
没有 wait()，notify() 会被忽略（无等待线程时调用 notify() 无副作用）  


## Thread.sleep(long millis)
- Thread类的静态方法
- 让线程等待一段时间后继续执行，不需要显式环境
```
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    e.printStackTrace();
}
```


## thread.join()
- Thread对象的实例方法
- 等待线程执行结束
- 让主线程等待该线程执行完毕后再继续执行
```
Thread thread = new Thread(() -> {
    try {
        Thread.sleep(2000);
        System.out.println("子线程执行完毕");
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
});

thread.start();
System.out.println("主线程等待子线程结束...");
thread.join(); // 主线程阻塞，直到子线程结束
System.out.println("主线程继续执行");
```

## Thread.yield()
- Thread 类的静态方法
- 主动让出当前线程的CPU资源，让当前线程从 运行状态 转为 就绪状态，允许其他同优先级线程抢占 CPU
不释放锁，也不阻塞线程，仅提示调度器重新分配资源。  
无法保证一定生效（调度器可能忽略 yield() 请求）  
- 可以用于测试线程优先级等（强制触发线程上下文切换）
```
Thread highPriorityThread = new Thread(() -> {
    for (int i = 0; i < 3; i++) {
        System.out.println("高优先级线程执行：" + i);
        Thread.yield(); // 主动让出 CPU，观察低优先级线程是否能执行
    }
});
highPriorityThread.setPriority(Thread.MAX_PRIORITY);

Thread lowPriorityThread = new Thread(() -> {
    for (int i = 0; i < 3; i++) {
        System.out.println("低优先级线程执行：" + i);
    }
});
lowPriorityThread.setPriority(Thread.MIN_PRIORITY);

highPriorityThread.start();
lowPriorityThread.start();
```
- 可以降低后台不重要线程（比如异步日志）的CPU占用
```
public class LogTask implements Runnable {
    @Override
    public void run() {
        while (!Thread.currentThread().isInterrupted()) {
            // 执行轻量级任务（如日志记录）
            logInfo();
            Thread.yield(); // 让出 CPU，减少资源占用
        }
    }
}
```
- 可以降低线程空转的影响
```
while (conditionNotMet()) {
    Thread.yield(); // 让出 CPU，避免忙等待
}
```