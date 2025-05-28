---
title: 线程池的创建和运行逻辑
draft: false
weight: 2
---

# 线程池的创建和运行逻辑
## ThreadPoolExecutor
```
ThreadPoolExecutor executor = new ThreadPoolExecutor(
            2,                   // 核心线程数
            5,                   // 最大线程数
            60,                  // 空闲线程存活时间
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(10), // 任务队列
            Executors.defaultThreadFactory(), // 线程工厂
            new ThreadPoolExecutor.CallerRunsPolicy() // 拒绝策略
        );
```

### 主要参数
线程池内部使用队列+线程实现  
1. coolPoolSize:常驻核心线程数   
2. maxPoolSize:能够容纳同时执行的最大线程数   
3. keepAliveTime:多余的空闲线程的存活时间   
当线程池数量超过corePoolSize，空闲时间达到keepAliveTime，线程会被销毁到只剩下corePoolSize为止
4. handler：拒绝策略，当线程池超过max时，如何拒绝请求执行的runnable
5. workQueue:任务队列，提交但尚未被执行的任务
6. 线程工厂：线程工厂生产线程时可以指定线程的命名规则，可以设置线程分组，设置线程优先级等
   
   
使用线程池执行任务时：
1. 如果线程池的数量小于core，即使线程都空闲，也创建新线程处理任务
2. 线程池数量等于core，缓冲队列workQueue未满，任务放入任务队列等待
3. 大于core，队列满，数量小于max，创建新线程处理任务
4. 大于core，队列满，数量等于max，使用拒绝策略拒绝
5. 大于 core，空闲时间超过keep，线程被销毁到等于core

### 根据机器和应用设置线程数
```
# 获取CPU核数
int cpuCores = Runtime.getRuntime().availableProcessors();
# 计算密集型（主要消耗CPU）：线程数 = CPU核数 + 1
int corePoolSize = cpuCores + 1;
int maxPoolSize = cpuCores + 1;
# IO密集型（多IO等待）：线程数 = 2 * CPU核数
int corePoolSize = 2 * cpuCores;
int maxPoolSize = 2 * cpuCores;
```

### 拒绝策略
1. AbortPolicy  
拒绝任务直接抛出异常RejectedExecution(RuntimeException)，让调用者感知，根据业务选择重试或放弃
2. DiscardPolicy  
新任务提交后拒绝，不给通知。调用者不知道任务会被丢弃可能造成数据丢失
3. DiscardOldestPolicy  
丢弃任务队列的头节点（通常是存活时间最久的任务），腾出扣减给新提交的任务，也有数据丢失风险
4. CallerRunsPolicy  
把任务交给提交任务的线程执行。被提交的任务不会被丢弃，不会有业务损失；且提交任务的线程被占用，不会快速提交其他任务，减缓了任务提交的速度。线程池有空余时间处理其他任务，腾出空间

### 任务队列
1. SynchronousQueue  
直接提交形式的任务队列  
队列中不会有缓存的任务，每个任务已提交就会交给线程执行，如果线程数量达到最大值自然拒绝
2. ArrayBlockingQueue   
有界任务队列，可以指定队列大小
符合一般的线程池执行逻辑，任务到来先创建线程，达到corePoolSize则写入队列，队列满则继续创建线程消费任务，达到max则执行拒绝策略
3. LinkedBlockingQueue  
无界任务队列，不能指定队列大小，除非系统资源耗尽，否则不存在入队失败的情况。  
线程数达到corePoolSize不再增加，而是直接写入队列
4. PriorityBlockingQueue  
优先任务队列，无界队列  
前面几种队列都是先进先出，这种是后进先出，主张先完成最新提交的任务


## Executors
### Executors.newFixedThreadPool   
```
ExecutorService executor = Executors.newFixedThreadPool(3);
executor.submit(() -> {...});

# Java实现
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```
- 创建固定数量的线程池：核心线程数和最大线程数固定相等  
- 控制可并发的线程数，空闲线程会在队列中等待  
- 空闲线程不会被回收。
- 任务队列无界（LinkedBlockingQueue），可能导致 OOM
### Executors.newCachedThreadPool  
```
ExecutorService executor = Executors.newCachedThreadPool();

# Java实现
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```
- 核心线程数为 0，最大线程数为Integer.MAX_VALUE。
- 空闲线程存活 60 秒，自动回收。
- 适合短时间内大量任务的场景，可能导致资源耗尽 
### Executors.newSingleThreadExecutor 
```
ExecutorService executor = Executors.newSingleThreadExecutor();

# Java实现
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```
- 创建单线程线程池
- 始终只有一个工作线程。
- 任务按提交顺序串行执行。
- 保证所有任务按顺序执行，避免多线程并发问题
### Executors.newScheduledThreadPool 
```
ScheduledExecutorService executor = Executors.newScheduledThreadPool(2);

# Java实现
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}

# 使用
// a. scheduleAtFixedRate - 固定频率执行（任务开始时间间隔固定）
executor.scheduleAtFixedRate(
    () -> System.out.println("固定频率任务：" + System.currentTimeMillis()),
    0,         // 初始延迟
    2,         // 周期
    TimeUnit.SECONDS
);

// b. scheduleWithFixedDelay - 固定延迟执行（任务结束时间与下次开始时间间隔固定）
executor.scheduleWithFixedDelay(
    () -> {
        try {
            Thread.sleep(1500); // 模拟任务耗时
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("固定延迟任务：" + System.currentTimeMillis());
    },
    0,         // 初始延迟
    1,         // 延迟时间
    TimeUnit.SECONDS
);

// c. schedule - 延迟执行一次
executor.schedule(
    () -> System.out.println("仅执行一次的延迟任务"),
    5,         // 延迟时间
    TimeUnit.SECONDS
);

```
- 执行定时任务的线程池：支持定时或周期性任务执行
- scheduleAtFixedReat 固定频率开始任务，以上一次任务的开始时间和这次任务的开始时间间隔固定
- scheduleWithFixedDelay 固定延迟开始任务，以上一次任务的结束时间和这次任务的开始时间之间固定延迟
- sechedule 延迟一段时间执行

### Executors.newSingleThreadScheduledExecutor 
```
ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor();

# Java实现
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}

```
- 单线程的定时任务线程池，单线程保证任务按顺序执行
- 适合需要定时执行的顺序任务（如每日数据备份）

### Executors.newWorkStealingPool 
```
ExecutorService executor = Executors.newWorkStealingPool();

# Java实现
public static ExecutorService newWorkStealingPool() {
    return new ForkJoinPool
        (Runtime.getRuntime().availableProcessors(),
         ForkJoinPool.defaultForkJoinWorkerThreadFactory,
         null, true);
}
```
根据当前CPU情况生成线程池，适用计算密集型应用
- 基于ForkJoinPool实现，工作线程会从其他队列 “窃取” 任务。
- 并行度默认为 CPU 核心数，适合大计算量的异步任务。
- 任务队列使用Deque，减少线程竞争