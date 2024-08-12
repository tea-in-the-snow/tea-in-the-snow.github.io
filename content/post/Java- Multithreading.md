---
title: "Java中的多线程"
date: 2024-07-24 11:37:42+0800
comments: true
categories:
    - Programming Languages
tags:
    - Java
    - Multithreading
---

## 通过继承Thread类来创建线程

创建一个继承了java.lang.Thread类的类，重写run()方法。线程将在run()方法中开始它的生命。新建一个类的实例，调用start()方法来开始线程的执行。start()方法会调用run()方法。

```Java
public class MultithreadingDemo extends Thread {
    public void run() {
        try {
            System.out.println(Thread.currentThread().getName() + " is running.");
        }
        catch (Exception e) {
            e.printStackTrace();
        }
    }
}

public class Main {
    public static void main(String[] args) {
        int n = 6;
        for (int i = 0; i < n; i++) {
            MultithreadingDemo obj = new MultithreadingDemo();
            obj.start();
        }
    }
}
```

## 通过实现Runnable接口来创建线程

在创建Thread实例时，传入一个实现了Runnable接口的类的实例。

```Java
public class RunnableDemo implements Runnable {
    public void run() {
        try {
            System.out.println(Thread.currentThread().getName() + " is running.");
        }
        catch (Exception e) {
            e.printStackTrace();
        }
    }
}

public class Main {
    public static void main(String[] args) {
        int n = 6;
        for (int i = 0; i < n; i++) {
            Thread obj = new Thread(new RunnableDemo());
            obj.start();
        }
    }
}
```

## Java中线程的生命周期和状态

- New State: 新创建的线程，尚未运行。
- Runnable State: 在此状态下，线程可能正在运行，也可能随时准备运行。线程调度程序负责为线程提供运行时间。多线程程序为每个单独的线程分配固定的时间。每个线程运行一小段时间，然后暂停并将 CPU 交给另一个线程，以便其他线程有机会运行。发生这种情况时，所有准备运行、等待 CPU 的线程和当前正在运行的线程都处于Runnable状态。
- Blocked State: 当线程试图获取锁但当前锁已被其他线程获取时，该线程将处于Blocked状态。获取锁后，线程将从Blocked状态转为Runnable状态。
- Waiting State: 当线程调用wait()方法或join()方法时，它将处于Waiting状态。当其他线程通知或该线程终止时，它将转移到Runnable状态。
- Timed Waiting State: 当线程调用带有超时参数的方法时，它处于Timed Waiting状态。线程将一直处于此状态，直到超时完成或收到通知。例如，当线程调用sleep()或条件等待时，它将进入Timed Waiting状态。
- Terminated State: 线程由于以下任一原因而终止：当线程的代码已被程序完全执行时，线程正常退出；因为发生了一些不寻常的错误事件，例如分段错误或未处理的异常。

## Java线程的优先级

- public final int getPriority()方法返回线程的优先级，默认为5。
- public final void setPriority(int newPriority)方法设置线程的优先级，范围为1到10。

## 使用interrupt()方法中断线程

在其他线程中对要中断的目标线程调用interrupt()方法:

```java
public class MultithreadingDemo extends Thread {
    public void run() {
        try {
            System.out.println(Thread.currentThread().getName() + " is running.");
            int n = 0;
            while (!isInterrupted()) {
                n++;
                System.out.println(n + ". " + Thread.currentThread().getName() + " is running.");
            }
            System.out.println(Thread.currentThread().getName() + " is interrupted.");
        }
        catch (Exception e) {
            e.printStackTrace();
        }
    }
}

public class Main {
    public static void main(String[] args) throws InterruptedException {
        Thread newThread = new MultithreadingDemo();
        newThread.start();
        Thread.sleep(20);
        newThread.interrupt();
        newThread.join();
        System.out.println("end.");
    }
}
```

interrupt()方法向线程newThread发出了中断请求，线程要响应中断请求，需要使用isInterrupt()方法进行检测。

如果某个线程处于等待状态(Waiting State)，例如调用join()或者sleep()方法等待时，对线程调用interrupt()方法，join()或者sleep()方法会立刻抛出InterruptedException。

```java
public class RunnableDemo implements Runnable {
    public void run() {
        Thread.currentThread().setName("threadTwo");
        System.out.println(Thread.currentThread().getName() + " is running.");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + " is finished.");
    }
}

public class MultithreadingDemo extends Thread {
    public void run() {
        Thread.currentThread().setName("newThread");
        System.out.println(Thread.currentThread().getName() + " is running.");
        Thread threadTwo = new Thread(new RunnableDemo());
        threadTwo.start();
        try {
            threadTwo.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + " is finished.");
    }
}

public class Main {
    public static void main(String[] args) throws InterruptedException {
        Thread newThread = new MultithreadingDemo();
        newThread.start();
        Thread.sleep(20);
        newThread.interrupt();
        System.out.println("end.");
    }
}
```

上面的代码最终输出如下：

```shell
newThread is running.
threadTwo is running.
end.
newThread is finished.
java.lang.InterruptedException
    at java.base/java.lang.Object.wait0(Native Method)
    at java.base/java.lang.Object.wait(Object.java:375)
    at java.base/java.lang.Thread.join(Thread.java:2046)
    at java.base/java.lang.Thread.join(Thread.java:2122)
    at MultithreadingDemo.run(MultithreadingDemo.java:8)
threadTwo is finished.
```

首先，Main线程新建线程newThread，newThread新建线程threadTwo并调用join()方法等待threadTwo线程结束。Main线程对newThread线程调用interrupt()方法请求newThread线程终止，此时newThread方法还在等待中，会结束等待并且抛出InterruptedException。而threadTwo线程不受影响。

## volatile关键字

volatile关键字修饰变量可以保证变量的可见性，即一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

在Java虚拟机中，变量的值保存在主内存中，但是，当线程访问变量时，它会先获取一个副本，并保存在自己的工作内存中。如果线程修改了变量的值，虚拟机会在某个时刻把修改后的值回写到主内存，但是，这个时间是不确定的。

这会导致如果一个线程更新了某个变量，另一个线程读取的值可能还是更新前的，这就造成了多线程之间共享的变量不一致。

因此，volatile关键字的目的是告诉虚拟机：每次访问变量时，总是获取主内存的最新值；每次修改变量后，立刻回写到主内存。

```java
public class MultithreadingDemo extends Thread {
    public volatile int test = 0;
    public void run() {
        Thread.currentThread().setName("newThread");
        System.out.println(Thread.currentThread().getName() + " is running.");
        int n = 0;
        while (test == 0) {
            n++;
            System.out.println("n = " + n);
        }
        System.out.println(Thread.currentThread().getName() + " is finished.");
    }
}

public class Main {
    public static void main(String[] args) throws InterruptedException {
        MultithreadingDemo newThread = new MultithreadingDemo();
        newThread.start();
        Thread.sleep(10);
        newThread.test = 1;
        System.out.println("end.");
    }
}
```

## 线程池

线程过多会带来额外的开销，其中包括创建销毁线程的开销、调度线程的开销等等，同时也降低了计算机的整体性能。线程池维护多个线程，等待监督管理者分配可并发执行的任务。这种做法，一方面避免了处理任务时创建销毁线程开销的代价，另一方面避免了线程数量膨胀导致的过分调度问题，保证了对内核的充分利用。同时，也能够提高响应速度，任务到达时，无需等待线程创建即可立即执行。

### 池化思想

为了最大化收益并最小化风险，而将资源统一在一起管理的一种思想。

例如：内存池：预先申请内存，提升申请内存速度，减少内存碎片。连接池：预先申请数据库连接，提升申请连接的速度，降低系统的开销。实例池：循环使用对象，减少资源在初始化和释放时的昂贵损耗。

### 总体设计

线程池的核心实现类是ThreadPoolExecutor，实现的顶层接口是Executor。顶层接口Executor提供了一种思想：将任务提交和任务执行进行解耦。用户无需关注如何创建线程，如何调度线程来执行任务，用户只需提供Runnable对象，将任务的运行逻辑提交到执行器(Executor)中，由Executor框架完成线程的调配和任务的执行部分。

线程池在内部实际上构建了一个生产者消费者模型，将线程和任务两者解耦，并不直接关联，从而良好的缓冲任务，复用线程。线程池的运行主要分成两部分：任务管理、线程管理。任务管理部分充当生产者的角色，当任务提交后，线程池会判断该任务后续的流转：（1）直接申请线程执行该任务；（2）缓冲到队列中等待线程执行；（3）拒绝该任务。线程管理部分是消费者，它们被统一维护在线程池内，根据任务请求进行线程的分配，当线程执行完任务后则会继续获取新的任务去执行，最终当线程获取不到任务的时候，线程就会被回收。

### 任务调度

所有任务的调度都是由execute方法完成的，这部分完成的工作是：检查现在线程池的运行状态、运行线程数、运行策略，决定接下来执行的流程，是直接申请线程执行，或是缓冲到队列中执行，亦或是直接拒绝该任务。其执行过程如下：

- 首先检测线程池运行状态，如果不是RUNNING，则直接拒绝，线程池要保证在RUNNING的状态下执行任务。
- 如果workerCount < corePoolSize，则创建并启动一个线程来执行新提交的任务。
- 如果workerCount >= corePoolSize，且线程池内的阻塞队列未满，则将任务添加到该阻塞队列中。
- 如果workerCount >= corePoolSize && workerCount < maximumPoolSize，且线程池内的阻塞队列已满，则创建并启动一个线程来执行新提交的任务。
- 如果workerCount >= maximumPoolSize，并且线程池内的阻塞队列已满, 则根据拒绝策略来处理该任务, 默认的处理方式是直接抛异常。

### 简单实践

```java
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class Main {
    public static void main(String[] args)  {
        // 新建线程池
        ThreadPoolExecutor threadPoolExcutor = new ThreadPoolExecutor(5, 10, 5, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>());
        for (int i = 0; i < 15; i++) {
            // lambda表达式中变量需要是final或有效final
            int n = i + 1;
            // 向线程池中提交任务
            threadPoolExcutor.submit(() -> {
                try {
                    System.out.println("Starting " + Thread.currentThread().getName() + ". Executing task " + n);
                    Thread.sleep(5000);
                    System.out.println("Ending " + Thread.currentThread().getName() + ". Finishing task " + n);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
        // 关闭线程池，在之前提交的任务将会被执行完，但线程池不会接受新任务。
        // 但是这个方法不会等待线程池接受的任务执行完毕，需要使用awaitTermination来做到这一点。
        threadPoolExcutor.shutdown();
        System.out.println("Main thread exiting");
    }
}
```

执行结果如下：

```shell
Main thread exiting
Starting pool-1-thread-5. Executing task 5
Starting pool-1-thread-1. Executing task 1
Starting pool-1-thread-2. Executing task 2
Starting pool-1-thread-3. Executing task 3
Starting pool-1-thread-4. Executing task 4
Ending pool-1-thread-3. Finishing task 3
Ending pool-1-thread-5. Finishing task 5
Starting pool-1-thread-3. Executing task 6
Ending pool-1-thread-2. Finishing task 2
Ending pool-1-thread-1. Finishing task 1
Starting pool-1-thread-5. Executing task 7
Ending pool-1-thread-4. Finishing task 4
Starting pool-1-thread-2. Executing task 8
Starting pool-1-thread-1. Executing task 9
Starting pool-1-thread-4. Executing task 10
Ending pool-1-thread-4. Finishing task 10
Ending pool-1-thread-5. Finishing task 7
Starting pool-1-thread-5. Executing task 12
Ending pool-1-thread-3. Finishing task 6
Starting pool-1-thread-3. Executing task 13
Ending pool-1-thread-1. Finishing task 9
Ending pool-1-thread-2. Finishing task 8
Starting pool-1-thread-4. Executing task 11
Starting pool-1-thread-1. Executing task 14
Starting pool-1-thread-2. Executing task 15
Ending pool-1-thread-5. Finishing task 12
Ending pool-1-thread-1. Finishing task 14
Ending pool-1-thread-4. Finishing task 11
Ending pool-1-thread-3. Finishing task 13
Ending pool-1-thread-2. Finishing task 15
```

## ThreadLocal

ThreadLocal让我们能够存储只能被特定线程保存的数据。

例如：如果我们想要有存储一个与特定线程绑定的Integer
