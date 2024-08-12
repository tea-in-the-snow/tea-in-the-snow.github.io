---
title: "Java synchronized 关键字"
date: 2024-07-27 11:37:42+0800
comments: true
categories:
    - Programming Languages
tags:
    - Java
---

## synchronized 关键字

- synchronized lock：同步锁

## 修饰实例方法

在方法声明中添加synchronized关键字：

```Java
public synchronized void synchronisedCalculate() {
    setSum(getSum() + 1);
}
```

该类的每个实例只有一个线程能够执行这个方法。被synchronized修饰函数简称同步函数，线程执行称同步函数前，需要先获取监视器锁，简称锁，获取锁成功才能执行同步函数，同步函数执行完后，线程会释放锁并通知唤醒其他线程获取锁，获取锁失败则阻塞并等待通知唤醒该线程重新获取锁，同步函数会以this作为锁，即当前对象。

## 修饰静态方法

```Java
 public static synchronized void syncStaticCalculate() {
     staticSum = staticSum + 1;
 }
```

每个类只有一个线程能够执行这个方法。使用Synchronized的方式与普通方法一致，唯一的区别是锁的对象不再是this，而是Class对象。

## 修饰代码块

```Java
public void performSynchronisedTask() {
    synchronized (this) {
        setCount(getCount()+1);
    }
}
```

this是向代码块传递的参数。synchronized能够接受任何对象作为锁，对于每一个对象，只有一个线程能够在代码块中执行。