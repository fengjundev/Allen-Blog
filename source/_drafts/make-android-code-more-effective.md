---
title: 让Android代码更高效的技巧
date: 2016-12-14 20:18:15
tags: 
- android
- java
categories: 
- android
- java
---

本文主要从代码层面展开，着眼于可以提高App性能的代码编写小技巧。

## 避免创建不必要的对象
Android里面使用的是一个三级Generation的内存模型，最近分配的对象会存放在Young Generation区域，当这个对象在这个区域停留的时间达到一定程度，它会被移动到Old Generation，最后到Permanent Generation区域。每一个级别的内存区域都有固定的大小，此后不断有新的对象被分配到此区域，当这些对象总的大小快达到这一级别内存区域的阀值时，会触发GC的操作，以便腾出空间来存放其他新的对象。

通常来说，单个的GC并不会占用太多时间，但是大量不停的GC操作则会显著占用帧间隔时间(16ms)。如果在帧间隔时间里面做了过多的GC操作，那么自然其他类似计算，渲染等操作的可用时间就变得少了，最后对于可能就会产生丢帧等现象，用户的直观感受就是卡顿。

所以内存优化的一个方向是尽量减少内存的占用，对象的分配虽然不可避免，但我们仍然应该尽可能的减少。下面介绍一些常用的方法。

### 避免在循环或者onDraw等方法分配对象
大量对象在短时间内被创建可能导致Young Generation的内存区域达到阀值，从而触发GC。所以我们，应该避免在循环体的内部和一些调用非常频繁的方法中分配对象。

在for循环中，可以尝试把对象的创建移到循环体之外。

我们在创建自定义View的过程中，常常会重写`onDraw`方法：

```java
public class CircleView extends View {

    ...

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        RectF rectCircle = new RectF();
        Paint paintCircle;

        rectCircle.set(centerX-newRadius, centerY-newRadius, centerX+newRadius, centerY+newRadius);

        paintCircle = new Paint();
        paintCircle.setAntiAlias(true);
        paintCircle.setColor(Color.RED);
        paintCircle.setStyle(Paint.Style.FILL_AND_STROKE);

        canvas.drawOval(rectCircle, paintCircle);
    }

    ...
}
```

但是，由于每次屏幕发生绘制或者动画执行的过程中，`onDraw`方法都会被调用到，所以它的调用也是很频繁的。上述代码中会产生大量的`RectF`和`Paint`对象。一种更好的做法是将rectCircle和paintCircle提取出来作为独立的字段并提前初始化好：

```java
public class CircleView extends View {

    private RectF rectCircle;
    private Paint paintCircle;
    
    public CircleView(Context context) {
        super(context);
        
        rectCircle = new RectF();
        paintCircle = new Paint();

        rectCircle.set(centerX-newRadius, centerY-newRadius, centerX+newRadius, centerY+newRadius);

        paintCircle.setAntiAlias(true);
        paintCircle.setColor(Color.RED);
        paintCircle.setStyle(Paint.Style.FILL_AND_STROKE);
    }

    ...

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        canvas.drawOval(rectCircle, paintCircle);
    }

    ...
}
```

### 使用单例
单例（Singleton）使我们使用的最多的一种设计模式，使用这种模式可以只提供一个对象供全局使用。所以，使用单例也是避免创建不必要对象的一种方式。
单例的实现方式有很多，比如下面一种使用的比较多的：

```java
public class SingleInstance {
  private static volatile SingleInstance sInstance;
  private SingleInstance() {
  }
  
  public static SingleInstance getInstance() {
      if (null == sInstance) {
          synchronized (SingleInstance.class) {
              if (null == sInstance) {
                  sInstance = new SingleInstance();
              }
          }
      }
      return sInstance;
  }
}
```

但是，由于单例持有的静态变量sInstance会长期存在，这需要我们特别注意它可能带来的内存泄露问题。比如，当初始化单例需要Context对象的时候，应该尽可能的传入ApplicationContext而不是ActivityContext。

### 使用对象池技术
对象池是一种创建型设计模式。它会初始化一个固定大小的池子，我们每次创建对象时候先去池子中找有没有，如果有直接取出，没有则创建出来并返回给调用者。调用者使用后还到池子里，这样便可达到对象复用的目的。由此可见，使用对象池可以实现对象的缓存和复用，解决频繁创建与销毁的问题。

Android中广泛采用了对象池技术。比如Message、TypedArray、ThreadPoolExecutor等。以Message为例，使用下面三种方式都可以获得一个Message对象：
- new Message()
- Message.obtain()
- Handler.obtainMessage() (最后调用的也是Message.obtain())

由于因为系统对Message的管理使用了对象池技术，后面两种效率会明显高于第一种，

如果我们的应用需要频繁且大量的创建某个对象的时候，就可以考虑采用对象池技术。在Android中，我们可以使用`SynchronizedPool`来轻松实现一个对象池：
```java
public class BookPool {

    private static final Pools.SynchronizedPool<Book> sPool = new Pools.SynchronizedPool(10);

    public static Book obtain() {
        Book instance = sPool.acquire();
        return (instance != null) ? instance : new Book();
    }

    public void recycle() {
        sPool.release(this);
    }
    
    ...

}
```

可以看到使用非常方便，只需要在类内部创建一个SynchronizedPool，然后提供obtain和recycle接口即可。

当想创建Book的对象的时候，我们不再使用：

```java
Book book = new Book();
```

而是调用：

```java
Book book = BookPool.obtain();
```

当该对象不再使用的时候，记得要将其释放回对象池，否则对象池被用完后，以后每次再去obtain的话就会导致创建一个新对象，这样的话对象池就不再有任何作用。

```java
myObject.recycle();
```

### 合理设置ArrayList的初始大小
ArrayList是我们使用比较频繁的一个集合类，内部基于数组实现。当向ArrayList中添加一个元素是，如果现有数组长度不够，它会重新扩容，并采用System.arraycopy复制当前的数组添加到扩容后的数组后面。我们来看下其内部扩容的实现：

```java
public boolean add(E object) {
    Object[] a = array;
    int s = size;
    if (s == a.length) {
        Object[] newArray = new Object[s +
                (s < (MIN_CAPACITY_INCREMENT / 2) ?
                 MIN_CAPACITY_INCREMENT : s >> 1)];
        System.arraycopy(a, 0, newArray, 0, s);
        array = a = newArray;
    }
    a[s] = object;
    size = s + 1;
    modCount++;
    return true;
}
```

可以看到，扩容时，如果当前容量小于6，则让其等于12，否则扩大为原来的两倍。
也就是说，假如当前ArrayList的容量为100，那么它在需要再存储一个元素时，即第101个元素，由于容量不够进行一次扩容，容量就变为了200。而多出了一个元素就多了100个元素的空间，这太浪费内存资源了。此外，扩容还会导致整个数组进行一次内存复制，这也是需要占用一定资源的。

在Android中，ArrayList集合默认大小为12，合理的设置ArrayList的容量可避免集合进行扩容。


## 使用SparseArray等优化过的容器


## 避免对象数组
比如我们在自定义View的时候需要记录之前一些touch坐标的X、Y值，可以考虑使用两个int[]来保存这些信息，一个用于存储X坐标，另一个用于存储Y坐标标值。

## 避免使用枚举


## 使用static方法
如果某个类中的方法并不需要访问类中的任何字段，将之声明为静态方法可以提高15%到20%的调用速度。而且，这样做还有一个好处，使用者可以从方法签名中知道这个方法不能改变类的状态。


```java
class DemoClass {
	
}
```

## 使用static final来修饰常量
考虑下面的声明：
```java
static int mInt = 42;
static String mString = "Hello world!"
```
编译器会生成一个类初始化方法，叫`<clinit>`，这个方法会在类首次使用的时候调用。这个方法会把42存储进mInt中，并把类文件中的常量池中`"Hello world!"`的引用赋给mString。

这样会有一个问题：当这两个变量被使用的时候，他们需要经过一个变量查找的过程。

我们可以使用final关键字来优化：

```java
static final int mInt = 42;
static final String mString = "Hello world!"
```
这样，类便不需要一个`<clinit>`方法，因为这些常量会在dex文件的静态初始化，使用mInt的代码会直接被42这个值替代，使用mString的代码会转为一个相对代价更低的方式：“string constant”


> 注意：这种优化方式适用于基本类型和String类型常量，并不是所有类型都适用。

## 避免在内部使用Getter和Setter
在Android中，非静态方法的调用在代价是昂贵的，比上一条所说的类文件实例查找更为耗时。当然，由于Java是一种面向对象的语言，在一些公有的类中提供getter和setter是一个很好地实践，因为我们可以通过这两个方法去对变量进行加工，但在类内部需要访问自己的字段时，直接访问这个字段会更加高效。

没有JIT(Just In Time Complier)时，对字段的直接访问会比使用getter快3倍左右。
有JIT的时候，能快7倍以上。


如果使用了ProGuard的话，它会将内部对getter和setter的调用内联。

## 使用增强型for语句
增强型for语句就是我们常说的`for-each`语句，可以用在实现了`Iterable`接口的集合和数组中，如：

```java
for(Object o : objectList) {
	....
}
```


With collections, an iterator is allocated to make interface calls to hasNext() and next(). With an ArrayList, a hand-written counted loop is about 3x faster (with or without JIT), but for other collections the enhanced for loop syntax will be exactly equivalent to explicit iterator usage.


运行以下代码：
```java
static class Foo {
    int mSplat;
}

Foo[] mArray = ...

public void zero() {
    int sum = 0;
    for (int i = 0; i < mArray.length; ++i) {
        sum += mArray[i].mSplat;
    }
}

public void one() {
    int sum = 0;
    Foo[] localArray = mArray;
    int len = localArray.length;

    for (int i = 0; i < len; ++i) {
        sum += localArray[i].mSplat;
    }
}

public void two() {
    int sum = 0;
    for (Foo a : mArray) {
        sum += a.mSplat;
    }
}
```

```bash
结果。。。。
```

可以看到，

zero()是最慢的，因为它在循环中每次都需要获取mArray.length的值，JIT不知道该如何优化。
one()比上一个快一些，因为它预先把mArray.length存储在一个局部变量中，免去了每次通过mArray调用的耗时。但也只有在mArray.length这里提供了优化点。
two()是最快的，. It uses the enhanced for loop syntax introduced in version 1.5 of the Java programming language.

So, you should use the enhanced for loop by default, but consider a hand-written counted loop for performance-critical ArrayList iteration.

> Tip: Also see Josh Bloch's Effective Java, item 46.









