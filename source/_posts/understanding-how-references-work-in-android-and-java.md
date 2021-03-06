---
title: 理解Android中的引用类型
date: 2016-12-07 19:47:02
tags: 
- android
- java
categories: 
- android
- java
---

![](http://ww3.sinaimg.cn/large/006tNc79gw1falmalphm2j30dw09amy2.jpg)

Android中的对象有着4种引用类型，垃圾回收器对于不同的引用类型有着不同的处理方式，了解这些处理方式有助于我们避免写出会导致内存泄露的代码。

<!-- more -->

> 出处： *[Allen's Zone](http://allenfeng.com/)*
> 作者： *Allen Feng*

## 引用

首先我们要理解：什么是引用（reference）？

在Java中，一切都被视为对象，引用则是用来操纵对象的途径。

对象和引用之间的关系可以用遥控器（引用）来操纵电视机（对象）这个场景来理解。只要手持这个遥控器，就能保持与电视机的连接。当我们想要改变频道或者音量时，实际操控的是遥控器（引用），再由遥控器（引用）来调控电视机（对象），达到操控的目的。

来看一段代码：
```java
Car myCar = new Car(); 
myCar.run();
```

上面这句话的意思是，创建一个Car的对象，并将这个新建的对象的引用存储在myCar中，此时myCar就是用来操作这个对象的引用。当我们获得myCar，就可以使用这个引用去操作对象中的方法或者字段了。

注意，当我们尝试在一个未指向任何对象的引用上去操作对象时，就会遇到经典的空指针异常（NullPointerException）。可以理解成我们手持遥控器，房间里却没有电视机可与之对象（没有可以用来操控的对象）。

```java
Car myCar;
myCar.run();
```

## GC与内存泄露
Java的一个重要优点就是通过垃圾收集器(Garbage Collection，GC)自动管理内存的回收，开发者不需要通过调用函数来释放内存。在Java中，内存的分配是由程序分配的，而内存的回收是由GC来完成。
GC为了能够正确释放对象，会监控每一个对象的运行状态，包括对象的申请、引用、被引用、赋值等，GC都需要进行监控。监视对象状态是为了更加准确地、及时地释放对象，而**释放对象的根本原则就是该对象不再被引用**。

Android中采用了标注与清理（Mark and Sweep）回收算法：

> 从"GC Roots"集合开始，将内存整个遍历一次，保留所有可以被GC Roots直接或间接**引用**到的对象，而剩下的对象都当作垃圾对待并回收。

Android内存的回收管理策略可以用下面的过程来展示：

![](http://ww4.sinaimg.cn/large/006y8lVagw1fajbcgm81bj30ps0d4ab9.jpg)

![](http://ww2.sinaimg.cn/large/006y8lVagw1fajbcs57f1j30pi0cs0ty.jpg)

![图自Google I/O: Memory Management for Android Apps](http://ww2.sinaimg.cn/large/006y8lVagw1fajb52zgs4j30qa0d8myd.jpg)

上面三张图片描述了GC的遍历过程。
每个圆形节点代表一个对象（内存资源），箭头表示对象引用的路径（可达路径），黄色表示遍历后的当前对象与GC Roots存在可达路径。当圆形节点与GC Roots存在可达路径的时候，表示当前对象正在被使用，GC不会将其回收。反之，若圆形节点与GC Roots不存在可达路径，意味着这个对象不再被程序引用（蓝色节点），GC可以将之回收。

在Android中，每一个应用程序对应有一个单独的Dalvik虚拟机实例，而每一个Dalvik虚拟机的大小是固定的（如32M，可以通过`ActivityManager.getMemoryClass()`获得）。这意味着我们可以使用的内存不是无节制的。所以即使有着GC帮助我们回收无用内存，还是需要在开发过程中注意对内存的引用。否则，就会导致内存泄露。

结合上文所述，**内存泄露**指的是：
> 我们不再需要的对象资源仍然与GC Roots存在可达路径，导致该资源无法被GC回收。

Android中的对象有着4种引用类型，垃圾回收器对于不同的引用类型有着不同的处理方式，了解这些处理方式有助于我们避免写出会导致内存泄露的代码。

## Strong reference（强引用）
强引用我们最常用的一种引用类型。当我们使用`new`关键字去新建一个对象的时候，创建的就是强引用。

比如：
```java
MyObject object = new MyObject();
```
这段代码的意思是：一个新的MyObject对象被创建了，并且一个指向它的强引用被存储在object中。

当一个对象具有强引用，那么垃圾回收器是绝对不会的回收和销毁它的。对象的强引用可以在程序中到处传递。很多情况下，会同时有多个引用指向同一个对象。

强引用的存在限制了对象在内存中的存活时间。假如对象A中包含了一个对象B的强引用，那么一般情况下，对象B的存活时间就不会短于对象A。如果对象A没有显式的把对象B的引用设为null的话，就只有当对象A被垃圾回收之后，对象B才不再有引用指向它，才可能获得被垃圾回收的机会。

下面，我们举一个例子：

```java
public class MainActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        new MyAsyncTask(this).execute();
    }

    private class MyAsyncTask extends AsyncTask { 

        @Override
        protected Object doInBackground(Object[] params) {
            
            // 模拟耗时任务
        	try {
                Thread.sleep(60000);
        	} catch (InterruptedException e) {
            	e.printStackTrace();
        	}

            return doSomeStuff();
        }
        private Object doSomeStuff() {
            return new Object();
        }
        @Override
        protected void onPostExecute(Object object) {
            super.onPostExecute(object);
            // 更新UI
        }
    }
}
```

这段代码里，MyAsyncTask会跟随Activity的onCreate去创建并开始执行一个长时间的耗时任务，并在耗时任务完成后去更新MainActivity中的UI。这是一个很常见的使用场景，却会导致内存泄露问题:

在Java中，**非静态内部类会在其整个生命周期中持有对它外部类的强引用**。

MainActivity被销毁时，MyAsyncTask中的耗时任务可能仍没有执行完成，所以MyAsyncTask会一直存活。此时，由于MyAsyncTask持有着其外部类，即MainActivity的引用，将导致MainActivity不能被垃圾回收。如果MainActivity中还持有着Bitmap等大对象，反复进出这个页面几次可能就会出现OOM Crash了。

那么我们如何避免这样的问题出现呢？请看下文。

## WeakReference （弱引用）
弱引用通过类[WeakReference](https://developer.android.com/reference/java/lang/ref/WeakReference.html)来表示。弱引用并不能阻止垃圾回收。如果使用一个**强引用**的话，只要该引用存在，那么被引用的对象是不能被回收的。弱引用则没有这个问题。**在垃圾回收器运行的时候，如果对一个对象的所有引用都是弱引用的话，该对象会被回收。**

我们调整一下上面例子中的代码，使用弱引用去避免内存泄露：
```java
public class MainActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        new MyAsyncTask(this).execute();
    }

    private static class MyAsyncTask extends AsyncTask {
        private WeakReference<MainActivity> mainActivity;    
        
        public MyAsyncTask(MainActivity mainActivity) {   
            this.mainActivity = new WeakReference<>(mainActivity);            
        }
        @Override
        protected Object doInBackground(Object[] params) {

            // 模拟耗时任务
        	try {
                Thread.sleep(30000);
        	} catch (InterruptedException e) {
            	e.printStackTrace();
        	}
            return doSomeStuff();
        }
        private Object doSomeStuff() {
            //do something to get result
            return new Object();
        }
        @Override
        protected void onPostExecute(Object object) {
            super.onPostExecute(object);
            if (mainActivity.get() != null){
                // 更新UI
            }
        }
    }
}
```

大家可以注意到，主要的不同点在于，我们把MyAsyncTask改为了静态内部类，并且其对外部类MainActivity的引用换成了：
```java
private WeakReference<MainActivity> mainActivity;
```

修改之后，当MainActivity destroy的时候，由于MyAsyncTask是通过弱引用的方式持有MainActivity，所以并不会阻止MainActivity被垃圾回收器回收，也就不会有内存泄露产生了。

> *有同学可能会对此存疑：如果弱引用在MainActivity destroy之前（即MainActivity正常工作时）被回收，这样不就导致`mainActivity.get() == null`，无法更新UI了吗？*
> 
> 需要注意的是，**GC回收的是对象**，在垃圾回收器运行的时候，如果对一个对象的***所有引用***都是弱引用的话，该对象会被回收。
> 在MainActivity正常工作时，除了有`mainActivity`这个弱引用指向MainActivity，还会有其他强引用指向MainActivity（ActivityStack等）。所以，GC扫描的时候，对于MainActivity这个对象并非都是弱引用，GC Roots与MainActivity仍然是强可达（一个对象通过一系列强引用可到达）的，所以，此时通过`mainActivity.get()`并不会返回null.


## SoftReference（软引用）
我们可以把软引用理解成一种稍强的弱引用。使用类[SoftReference](https://developer.android.com/reference/java/lang/ref/SoftReference.html)来表示。

很多人可能会把弱引用和软引用搞混，注意他们的**区别在于**：如果一个对象只具有**软引用**，若内存空间足够，垃圾回收器就不会回收它；如果内存空间不足了，才会回收这些对象的内存。

而只具有**弱引用**的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。

所以从引用的强度来讲： 强引用 > 软引用 > 弱引用。

表面上看来，软引用非常适合于创建缓存。当系统内存不足的时候，缓存中的内容是可以被释放的。

但是，在实践中，**使用软引用作为缓存时效率是比较低的**，系统并不知道哪些软引用指向的对象应该被回收，哪些应该被保留。过早被回收的对象会导致不必要的工作，比如Bitmap要重新从SdCard或者网络上加载到内存。

所以使用软引用去缓存对象，虽然确实可以避免OOM问题，却不适用于某些场景。在Android开发中，一种更好的选择是使用[LruCache](https://developer.android.com/reference/android/util/LruCache.html)，LRU是Least Recently Used的缩写，即“最近最少使用”，它的内部会维护一个固定大小的内存，当内存不足的时候，会根据策略把最近最少使用的数据移除，让出内存给最新的数据。具体实现有兴趣的同学可以自行研究。

## PhantomReference（虚引用）
一个只被虚引用持有的对象可能会在**任何时候**被GC回收。虚引用对对象的生存周期完全没有影响，也无法通过虚引用来获取对象实例，仅仅能在对象被回收时，得到一个系统通知（只能通过是否被加入到ReferenceQueue来判断是否被GC，这也是唯一判断对象是否被GC的途径）。

我们都知道，java的Object类里面有个finalize方法，它的工作原理是这样的：一旦垃圾回收器准备好释放对象占用的内存空间，将首先调用其finalize方法，并且在下一次垃圾回收动作发生时，才会真正回收对象占用的内存。但是，问题在于，虚拟机不能保证finalize何时被调用，因为GC的运行时间是不固定的。

使用虚引用就可以解决这个问题，**虚引用主要用来跟踪对象被垃圾回收的活动**，主要用来实现比较精细的内存使用控制，这对于Android设备来说是很有意义的。比如，我们可以在确定一个Bitmap被回收后，再去申请另外一个Bitmap的内存，通过这种方式可以使得程序所消耗的内存维持在一个相对较低且稳定的水平。

虚引用的使用demo可以参考这篇文章：[How to use PhantomReference](http://android.amberfog.com/?p=470)

---

Refers: 
- [Google I/O: Memory Management for Android Apps](https://www.youtube.com/watch?v=_CruQY55HOk)
- [Android Reference](https://developer.android.com/reference/java/lang/ref/Reference.html)
- [Understanding Weak References Blog](https://community.oracle.com/blogs/enicholas/2006/05/04/understanding-weak-references)
- [How to use PhantomReference](http://android.amberfog.com/?p=470)






