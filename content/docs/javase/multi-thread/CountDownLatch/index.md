---
title: CountDownLatch
draft: false
weight: 7
---

# CountDownLatch
## 使用
new一个CountDownLatch，需要指定计数器值，表示需要等待执行完毕的线程数量   

开启多个线程执行任务，并调用CountDownLatch对象的await方法，开始等待任务执行  
每个线程的任务执行完后，都要调用CountDownLatch对象的countDown方法  
  
当调用countDown方法的线程数达到计数器值时，多线程任务执行完毕，await方法的等待状态被取消，外层逻辑继续向下执行  

```
CountDownLatch latch = new CountDownLatch(3);

// 创建并启动 3 个工作线程
for (int i = 0; i < 3; i++) {
    final int taskId = i;
    new Thread(() -> {
        try {
            System.out.println("任务" + taskId + "开始执行");
            // 执行...
            System.out.println("任务" + taskId + "完成");
        } catch (Exception e) {
            // ...
        } finally {
            // 任务完成后，计数器减 1
            latch.countDown();
        }
    }).start();
}

// 主线程等待，直到计数器变为 0
System.out.println("主线程等待所有任务完成...");
latch.await();
System.out.println("所有任务已完成，主线程继续执行");
```

## 原理
CountDownLatch基于AbstractQueuedSynchronized（AQS）实现  
await方法被调用时，主线程进入等锁队列  
countDown方法被调用时，CountDownLatch对象内部计数器-1  
CountDownLatch对象内部计数器为0时，主线程被唤醒，继续执行任务  