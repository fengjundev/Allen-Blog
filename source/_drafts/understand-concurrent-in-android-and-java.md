---
title: 理解Android中的并发（1）
date: 2016-12-24 15:22:52
tags: android
category: android
---


## 1. Java篇

选择并发的目的，大多数情况下，许多问题都可以通过顺序编程的但是去解决。然而，对于某些问题，若是能够并行地执行程序中的多个部分，则会变得非常方便甚至非常必要。

先要理解进程和线程的概念的区别。调度器的概念。线程原理。


### 线程的作用

### 线程的状态
新建。。。
图。。。


### Runnable
首先我们把可执行的操作定义成**任务**，线程可以用于驱动这些任务。在 Java 中的可以通过实现`Runnable`接口来提供。如：

```java
/**
 * Represents a command that can be executed. Often used to run code in a
 * different {@link Thread}.
 */
public interface Runnable {

    /**
     * Starts executing the active part of the class' code. This method is
     * called when a thread is started that has been created with a class which
     * implements {@code Runnable}.
     */
    public void run();
}
```

```java
public class CountDownTask implements interface {
	
	public CountDownTask() {

	}

    @Overide
	public void run() {
		....
	}
}
```

注意当你使用 Runnable 定义了一个任务后，虽然实现了 run 方法，但是要注意，直接调用这个 run 方法不会产生任何线程的能力。要实现线程的行为，我们必须把一个任务附着在线程上面。

### Thread
上面说到，任务由线程来驱动。在 java 中的我们使用 Thread 类来定义一个线程，下面展示了如何使用线程来启动一个任务：

```java
public void test() {
	Thread t = new Thread(new CountDownTask());
}
```

或者直接继承 Thread 实现自己的异步任务

实现 RUnnale 和继承 Thread 的区别？

介绍更深入一点。


我们还可以加入更多线程查看：

注意线程之间执行先后顺序的不确定性。

注意线程可能带来的内存泄露问题。


### Executor 
Java 通过 Executor（执行器）来为我们管理 Thread 对象，Executor在调用者和任务执行之间提供了一个间接层，它帮助我们管理异步任务的执行，而不需要显式地管理对象的生命周期。

直接 new THread 的弊端：
那你就out太多了，new Thread的弊端如下：
a. 每次new Thread新建对象性能差。
b. 线程缺乏统一管理，可能无限制新建线程，相互之间竞争，及可能占用过多系统资源导致死机或oom。
c. 缺乏更多功能，如定时执行、定期执行、线程中断。
相比new Thread，Java提供的四种线程池的好处在于：
a. 重用存在的线程，减少对象创建、消亡的开销，性能佳。
b. 可有效控制最大并发线程数，提高系统资源的使用率，同时避免过多资源竞争，避免堵塞。
c. 提供定时执行、定期执行、单线程、并发数控制等功能。

引入对象池、线程池的概念。

通过 Executor 提供了多种线程池：

引入`Executor`的定义。


例子：
```java
public class ThreadPoolDemo {
	
	public void run() {
		ExecutorService exec = Executor.newCachedThreadPool();
		for (int i = 0; i < 100; i++) {
			exec.execute(new CountDownTask(100));
		}
		exec.shutdown();
    }
}
```

查看下输出：
```bash
...
```

shutdown方法可以防止新任务再被提交给 Executor



- CachedThreadPool
在程序执行过程中通常会创建于所需数目相同数目的线程，然后在它


例子
使用场景
输出

- FixedThreadPool
例子
使用场景
输出


- SingleThreadExecutor
就像是线程数量为1的 FixedThreadPool，它还提供了一种重要的并发机制，即没有两个线程会被并发的调用。
如果向他提交了多个任务，这些任务会被排队。每个任务都会在同一个线程中依次执行。如下面的例子，每个任务都是被按照提交的顺序执行的。

例子
使用场景
输出



> 注意在任何线程池中，现有线程在可能的情况下，都有可能被自动复用。


### Callable
与 Runnable 相对，Callable 可以产生返回值。
从定义看

```java
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

这是一种类型参数的泛型，使用实例：

### Thread 常用方法
- Sleep
- join
- wait
- notify
- notifyAll
生产者消费者

### 线程的优先级

### 线程异常的捕获
UncaughtExceptionHandler

### 线程的同步
共享资源访问带来的问题
举例子、

### 死锁的概念
避免死锁？


浴室的例子。
### synchronized
synchronized直译。。
临界数据
Java以资源冲突的方式，为防止资源冲突提供了支持。
当任务要执行synchronized 保护的代码段或者方法时，它会先检查锁是否可用。

锁是什么？同步的对象是？

若不可用会一直等待。上一个释放锁，获取，执行，释放锁。攻下一个线程调用

具体使用方法。同步控制块，性能。单例

> 通过是使用同步控制块。。。。。而不是同步方法，性能提升

单例为例

对象的方法锁：
同一个对象枷锁，。。等上一个方法执行完，才可以
```java
```

什么时候使用synchronized？

### Lock 对象？？看情况介绍

### 原子访问
什么是原子操作
volatile 关键字


### 使用线程可能产生的问题
---

### 线程本地存储

### 线程的终止


### 并发相关数据结构

### 线程安全

### Java提供的并发库

## 2. Android篇

### Android 提供的线程形式
AsyncTask: 为 UI 线程与工作线程之间进行快速的切换提供一种简单便捷的机制。适用于当下立即需要启动，但是异步执行的生命周期短暂的使用场景。
HandlerThread: 为某些回调方法或者等待某些任务的执行设置一个专属的线程，并提供线程任务的调度机制。
ThreadPool: 把任务分解成不同的单元，分发到各个不同的线程上，进行同时并发处理。
IntentService: 适合于执行由 UI 触发的后台 Service 任务，并可以把后台任务执行的情况通过一定的机制反馈给 UI。

Refers:

[](http://www.trinea.cn/android/java-android-thread-pool/)
[](https://developer.android.com/reference/java/lang/Thread.html)
[](https://developer.android.com/guide/components/processes-and-threads.html)
[](https://developer.android.com/training/multiple-threads/define-runnable.html)
[](https://developer.android.com/training/multiple-threads/communicate-ui.html)
[](http://blog.csdn.net/a2011480169/article/details/52525363)
[](http://droidyue.com/blog/2016/03/13/learning-threadlocal-in-java/index.html)
[](http://arthennala.blog.51cto.com/287631/56356)
[](https://github.com/GeniusVJR/LearningNotes/blob/master/Part5/ReadingNotes/%E3%80%8A%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3java%E8%99%9A%E6%8B%9F%E6%9C%BA%E3%80%8B%E7%AC%AC12%E7%AB%A0.md)
[](http://www.importnew.com/19011.html)
[](http://mrpeak.cn/blog/android-threading/)

**! [](http://www.jianshu.com/p/d59b3cce2b54)

[](http://wangzhaoli.blog.51cto.com/7607113/1287545)

[](https://realm.io/cn/news/android-threading-background-tasks/)

**! [](http://bugly.qq.com/bbs/forum.php?mod=viewthread&tid=1022)

**! [](http://www.infoq.com/cn/articles/android-worker-thread)



---

---


volatile比同步更简单，只适用于控制对基本变量（整数、布尔值等）的单个实例的访问。当一个变量被声明成为volatile的时候，任何对这个变量的写入操作或者读取操作都将直接绕过高速缓存，直接写入或者读取主内存。

这表示所有线程看到的volatile变量都将相同。

valatile对于确保每个线程看到最新的变量值非常有用，但是有时我们需要保护比较大的代码片段。

这个时候，synchronized就派上用场了。

同步使用监控器（monitor）或者锁的概念，协调对特定代码的访问。







---

---

---

*Java Concurrency in Practice*

共享以为着变量可以被多个线程同时访问，而“可变”意味着变量的值在其生命周期内可以发生变化


对象是否线程安全，取决于是否会被多个线程访问，而非对象本身要实现的功能

要使得对象线程安全，就需要使用同步机制来协同这些线程对变量的访问。


原子性：i++，包含读取，修改，写入三个步骤









































