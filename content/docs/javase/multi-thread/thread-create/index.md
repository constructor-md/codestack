---
title: 线程的创建和使用
draft: false
weight: 1
---

# 线程的创建和使用

可以直接new Thread()创建一个线程  
要想让线程执行某个任务，创建线程时可以传入具体的一个Runnable对象，内含实现方法  
最后调用线程的start方法开始执行  

设置任务的方式：
## 实现Runnable接口
内部实现run方法，创建线程时作为实例化参数传递进去，则start线程后会执行run方法  
无法获取返回值，run方法是void的，且不支持传递参数进去  
```
public class RunnableExample {
    public static void main(String[] args) {
        // 创建Runnable接口的实现类
        Runnable task = new Runnable() {
            @Override
            public void run() {
                System.out.println("通过实现Runnable接口创建的线程正在执行任务");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("任务执行完毕");
            }
        };

        // 创建线程并传入Runnable实例
        Thread thread = new Thread(task);
        // 启动线程
        thread.start();
    }
}
```
## 实现Callable接口
这是一个泛型接口，实现其中的call方法，方法的返回值即接口指定的泛型  
实现类放进一个FutureTask对象中，将FutureTask对象传入Thread构造方法  
执行完毕后，通过FutureTask对象的get方法可以获取返回值  
```
import java.util.concurrent.*;

public class CallableExample {
    public static void main(String[] args) {
        // 创建Callable接口的实现类
        Callable<String> callableTask = new Callable<String>() {
            @Override
            public String call() throws Exception {
                System.out.println("通过实现Callable接口创建的线程正在执行任务");
                Thread.sleep(2000);
                return "任务执行结果：成功";
            }
        };

        // 将Callable包装成FutureTask
        FutureTask<String> futureTask = new FutureTask<>(callableTask);
        // 创建线程并传入FutureTask
        Thread thread = new Thread(futureTask);
        // 启动线程
        thread.start();

        try {
            // 获取任务执行结果（会阻塞直到任务完成）
            String result = futureTask.get();
            System.out.println("获取到的返回值：" + result);
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```
## 继承Thread类
因为Thread类实现了Runnable接口，我们可以继承Thread类实现run方法，实例化我们实现的子类，然后执行start，即可运行其中的方法  
但是继承整个Thread类开销过大，一般不使用。  
且Java不支持多重继承，继承了Thread接口无法继承其他类，但是接口则可以实现多个，所以使用接口实现传递方法比较好。  
```
public class ThreadSubclassExample {
    public static void main(String[] args) {
        // 创建自定义线程类的实例
        CustomThread customThread = new CustomThread();
        // 启动线程
        customThread.start();
    }

    // 继承Thread类并重写run方法
    static class CustomThread extends Thread {
        @Override
        public void run() {
            System.out.println("通过继承Thread类创建的线程正在执行任务");
            try {
                Thread.sleep(1500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("任务执行完毕");
        }
    }
}
```
