---
title: 理解Android中的LayoutInflater
date: 2017-02-24 14:54:56
tags:
- android
- 布局加载
- 源码解析
- LayoutInflater
categories: 
- android
- 源码解析
---

![](https://ww3.sinaimg.cn/large/006y8lVagy1fd32cpa79kj30dq096q3f.jpg)

大家对LayoutInflater一定不陌生，它主要用于加载布局，在Fragment的**onCreateView**方法、ListView Adapter的getView方法等许多地方都可以见到它的身影。今天主要聊聊LayoutInflater的用法以及加载布局的工作原理。

<!-- more -->

> 出处： *[Allen's Zone](http://allenfeng.com/)*
> 作者： *Allen Feng*

## 什么是LayoutInflater

LayoutInflater是一个**用于将xml布局文件加载为View或者ViewGroup对象的工具**，我们可以称之为**布局加载器**。

## 用法

### 获取LayoutInflater

首先要注意LayoutInflater本身是一个抽象类，我们不可以直接通过`new`的方式去获得它的实例，通常有下面三种方式：

**第一种：**
```java
LayoutInflater inflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
```

**第二种：**
```java
LayoutInflater inflater = LayoutInflater.from(context); 
```

**第三种：**
```java
在Activity内部调用getLayoutInflater()方法
```

看看后面两种方法的实现：


```java
public static LayoutInflater from(Context context) {
    LayoutInflater LayoutInflater =
            (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    if (LayoutInflater == null) {
        throw new AssertionError("LayoutInflater not found.");
    }
    return LayoutInflater;
}
```

在Activity内部调用getLayoutInflater方法其实调用的是PhoneWindow的mLayoutInflater:
```java
public PhoneWindow(Context context) {
    super(context);
    mLayoutInflater = LayoutInflater.from(context);
}
```


所以，这几个方法实际上殊途同归，都是通过调用Context的getSystemService方法去获取。获取到的是`PhoneLayoutInflater`这个实现类，具体的获取过程就不在这里展开分析了。

```java
public class Policy implements IPolicy {
	...
    public LayoutInflater makeNewLayoutInflater(Context context) {
        return new PhoneLayoutInflater(context);
    }
}    
```

### 加载布局
我们用一个简单的例子，介绍下LayoutInflater的用法：

> 这个例子的目标是在屏幕上展示一个按钮，点击按钮时，会通过LayoutInflater把一个**橙色背景**的TextView以`match_parent`的形式加载到一块宽高为300dp的RelativeLayout中。

首先创建两个布局文件：

**demo_layout.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<TextView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:layout_gravity="center"
    android:gravity="center"
    android:background="#ff750c"
    android:text="Hello , world !" />
```

**activity_main.xml**
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <Button
        android:id="@+id/button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerHorizontal="true"
        android:text="点击加载" />

    <RelativeLayout
        android:id="@+id/root"
        android:layout_width="300dp"
        android:layout_height="300dp"
        android:layout_centerInParent="true"/>


</RelativeLayout>

```

**MainActivity.java**
```java
public class MainActivity extends Activity {

    RelativeLayout rootView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        rootView = (RelativeLayout) findViewById(R.id.root);

        findViewById(R.id.button).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                inflateView();
            }
        });
    }

    private void inflateView() {

        View insideView = LayoutInflater.from(MainActivity.this).inflate(R.layout.demo_layout, null);
        rootView.addView(insideView);
    }
}
```

编译运行，点击`点击加载`按钮，结果如下：

![点击按钮后](https://ww2.sinaimg.cn/large/006y8lVagy1fd3lge1629j30900g00su.jpg)



可以看到，我们成功把**demo_layout.xml**对应布局中的TextView加载进来了。

但遗憾的是，加载进来的TextView宽高**并不是我们期望的300x300大小**╮(╯▽╰)╭。

那么问题来了：
> 
> - 为什么我们在布局文件中给TextView设置的宽高属性失效了呢？
> - LayoutInflater又是如何把xml解析加载成为View的呢？

![](https://ww1.sinaimg.cn/large/006y8lVagy1fd3kx9kl4zg30280280tu.gif)


而且，inflate有多个不同的重载方法：
1. `inflate(int resource, ViewGroup root)`
2. `inflate(int resource, ViewGroup root, boolean attachToRoot)`
3. `inflate(XmlPullParser parser, ViewGroup root)`
4. `inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot)`

上面的例子只是使用了第一个方法，并且**root**的传参还是null。

这些方法又有什么不同的地方呢？

我们从源码入手，去寻找这两个问题的答案。

## 源码解析

上文有提到，我们获取的LayoutInflater实例其实是PhoneLayoutInflater，但PhoneLayoutInflater并没有重写inflate的几个方法，所以我们的分析还是在LayoutInflater这个类展开。

首先比对下这几个重载方法：

```java
public View inflate(int resource, ViewGroup root) {
    // root不为空时，attachToRoot默认为true
    return inflate(resource, root, root != null);
}

public View inflate(int resource, ViewGroup root, boolean attachToRoot) {
    XmlResourceParser parser = getContext().getResources().getLayout(resource);
    try {
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}

public View inflate(XmlPullParser parser, ViewGroup root) {
    // root不为空时，attachToRoot默认为true
    return inflate(parser, root, root != null);
}

public View inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot) {
	...
}
```

原来，前三个方法最终调用的都是:
```java
public View inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot) {
	...
}
```
而且，**root不为空时，attachToRoot默认为true**。布局id会被通过调用`getLayout`方法生成一个`XmlResourceParser`对象。

Android中布局文件都是使用xml编写的，所以解析过程自然涉及xml的解析。常用的xml解析方式有DOM，SAX和PULL三种方式。DOM不适合xml文档较大，内存较小的场景，所以不适用于手机这样内存有限的移动设备上。SAX和PULL类似，都具有解析速度快，占用内存少的优点，而相对之下，PULL的操作方式更为简单易用，所以，Android系统内部在解析各种xml时都用的是PULL解析器。

这里解析布局xml文件时使用的就是Android系统提供的PULL方式。

我们继续分析inflate方法。

### inflate方法
```java
public View inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            
            final AttributeSet attrs = Xml.asAttributeSet(parser);

            // 首先注意result初值为root
            View result = root;

            try {
            	// 尝试找到布局文件的根节点
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }

                ...

                // 获取当前节点名称，如merge，RelativeLayout等
                final String name = parser.getName();
                
                ...

                // 处理merge节点
                if (TAG_MERGE.equals(name)) {

                	// merge必须依附在一个根View上
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }

                    rInflate(parser, root, attrs, false);

                } else {
                    
                    View temp;
                    
                    // 根据当前信息生成一个View
                    temp = createViewFromTag(root, name, attrs);
                    ...

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                       
                        // 如果指定了root参数的话，根据节点的布局参数生成合适的LayoutParams
                        params = root.generateLayoutParams(attrs);

                        // 若指定了attachToRoot为false，会将生成的布局参数应用于上一步生成的View
                        if (!attachToRoot) {
                            temp.setLayoutParams(params);
                        }
                    }

                    // 由上至下，递归加载xml内View，并添加到temp里
                    rInflate(parser, temp, attrs, true);

                    // 如果root不为空且指定了attachToRoot为true时，会将temp作为子View添加到root中
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // 如果指定的root为空，或者attachToRoot为false的时候，返回的是加载出来的View，
                    // 否则返回root
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } ... // 异常处理

            return result;
        }
    }
```


首先定义**布局根View**这一个概念，注意与root并不是同一个东西：
- root是我们传进来的第二个参数
- 布局根View则是传递进来的布局文件的**根节点所对应的View**

这个方法主要有下面几个步骤：
1. 首先查找根节点，如果整个xml文件解析完毕也没看到根节点，会抛出异常；
2. 如果查找到的根节点名称是merge标签，会调用**rInflate**方法继续解析布局，最终返回root；
3. 如果是其他标签（View、TextView等），会调用**createViewFromTag**生成布局根View，并调用**rInflate**递归解析余下的子View，添加至布局根View中，最后视root和attachToRoot参数的情况最终返回view或者root。


从这里我们可以理清root和attachToRoot参数的关系了：

- `root != null， attachToRoot == true：`
传进来的布局会被加载成为一个View并作为子View添加到root中，最终返回root；
而且这个布局根节点的android:layout_参数会被解析用来设置View的大小。

- `root == null， attachToRoot无用：`
当root为空时，attachToRoot是什么都没有意义，此时传进来的布局会被加载成为一个View并直接返回；
布局根View的android:layout_xxx属性会被忽略。

- `root != null， attachToRoot == false：`
传进来的布局会被加载成为一个View并直接返回。
布局根View的android:layout_xxx属性会被解析成LayoutParams并保留。(root只用来参与生成布局根View的LayoutParams)


现在可以解答文章开始留下的疑问了：
**为何在布局文件中给TextView设置的android:layout属性失效了?**

回到例子中的代码，我们加载布局的代码是:

```java
View insideView = LayoutInflater.from(MainActivity.this).inflate(R.layout.demo_layout, null);
rootView.addView(insideView);
```

即root传参为空，与上面第2种情况对应，所以此时布局根View的android:layout_xx属性都被忽略了。也就是相当于并没有给TextView设置宽高，所以只能按默认的TextView大小显示了。

稍微改变下代码：
```java
LayoutInflater.from(MainActivity.this).inflate(R.layout.demo_layout, rootView);
```

注意这段代码等同于：
```java
LayoutInflater.from(MainActivity.this).inflate(R.layout.demo_layout, rootView, true);
```

>inflate方法在root不为空时，默认会将attachToRoot置为true。

这时等同于我们上面的情况1，由于此时infalte会将加载出来的View自动添加到root中，我们要把`rootView.addView(insideView)`一句移除，否则会遇到这样的报错：
```bash
 java.lang.IllegalStateException: The specified child already has a parent. You must call removeView() on the child's parent first.
			at android.view.ViewGroup.addViewInner(ViewGroup.java:4454)
			at android.view.ViewGroup.addView(ViewGroup.java:4295)
			at android.view.ViewGroup.addView(ViewGroup.java:4235)
			at android.view.ViewGroup.addView(ViewGroup.java:4208)
```

再来运行看看：

![](https://ww4.sinaimg.cn/large/006y8lVagy1fd3sf1v7dmj30900g03ym.jpg)

终于达到我们想要的效果了！这也验证了上面的第一个结论。

顺便再用这个例子拓展一下，验证我们的情况3，即`root != null， attachToRoot == false`时的情况：

```java
View insideView = LayoutInflater.from(MainActivity.this).inflate(R.layout.demo_layout, rootView, false);
rootView.addView(insideView);
```

结果是一样的，图就不贴了，即`root != null， attachToRoot == false`时，root只是用来参与布局根View的大小、位置设置的。

好了，关于这两个参数的疑问的解答就告一段落了，我们接着回到代码，寻找另一个问题的答案。
继续跟进**rInflate**和**createViewFromTag**方法。

### rInflate方法

```java
void rInflate(XmlPullParser parser, View parent, final AttributeSet attrs,
        boolean finishInflate) throws XmlPullParserException, IOException {

    ...

    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

        ...

        final String name = parser.getName();
        
        // 解析“requestFocus”标签，让父View调用requestFocus()获取焦点
        if (TAG_REQUEST_FOCUS.equals(name)) {
            ...
        } else if (TAG_INCLUDE.equals(name)) {
        	...

        } else if (TAG_MERGE.equals(name)) {
        	...
        } else if (TAG_1995.equals(name)) {
            ...      
        } else {

        	// 调用createViewFromTag生成一个View
            final View view = createViewFromTag(parent, name, attrs);

            // 逐层递归调用rInflate，解析view嵌套的子View
            rInflate(parser, view, attrs, true);

            // 将解析生成子View添加到上一层View中
            viewGroup.addView(view, params);
        }
    }

    // 内层子View被解析出来后，将调用其父View的“onFinishInflate()”回调
    if (finishInflate) parent.onFinishInflate();
}
```

首先是几个特殊标签的处理，如`requestFocus`、`include`等，为了把握住主要脉络，我们不做展开，直接看最后一个else的内容。

原来，rInflate主要是调用了createViewFromTag生成当前解析到的View节点，并**递归调用rInflate逐层生成子View**，添加到各自的上层View节点中。
当某个节点下面的所有子节点View解析生成完成后，才会调起onFinishInflate回调。

所以createViewFromTag才是真正生成View的地方啊。

### createViewFromTag方法

```java
View createViewFromTag(View parent, String name, AttributeSet attrs) {

    ...

    try {

        View view;
        if (mFactory2 != null) view = mFactory2.onCreateView(parent, name, mContext, attrs);
        else if (mFactory != null) view = mFactory.onCreateView(name, mContext, attrs);
        else view = null;

        if (view == null && mPrivateFactory != null) {
            view = mPrivateFactory.onCreateView(parent, name, mContext, attrs);
        }
        
        // 三个Factory都不存在调用LayoutInflater自己的onCreateView或者createView
        // 
        // 如果View标签中没有"."，则代表是系统的widget，则调用onCreateView，
        // 这个方法会通过"createView"方法创建View
        // 不过前缀字段会自动补"android.view."前缀。
        if (view == null) {
            if (-1 == name.indexOf('.')) {
                view = onCreateView(parent, name, attrs);
            } else {
                view = createView(name, null, attrs);
            }
        }

        return view;

    } catch (InflateException e) {
        ...
    } ...
}

public interface Factory {
    public View onCreateView(String name, Context context, AttributeSet attrs);
}
```

首先会依次调用mFactory2、mFactory和mPrivateFactory三者之一的onCreateView方法去创建一个View。
如果这几个Factory都为null，会调用LayoutInflater自己的onCreateView或者createView来实例化View。


> 自定义Factory一个十分有用的使用场景就是实现**应用换肤**，有兴趣的读者可以参考我开源的[Android-Skin-Loader](https://github.com/fengjundev/Android-Skin-Loader)中的具体细节。


通常情况下，自定义工厂mFactory2、mFactory和私有工厂mPrivateFactory是空的，当Activity继承自AppCompatActivity时，才会存在自定义Factory。

所以，生成View的重任就落在了**onCreateView**和**createView**身上。


**onCreateView**调用的其实是**createView**，即View的节点名称没有`.`时，将自动补上`android.view.`前缀（即完整类名）：
```java
protected View onCreateView(String name, AttributeSet attrs)
        throws ClassNotFoundException {
    return createView(name, "android.view.", attrs);
}
```

继续关注的**createView**实现。

### createView方法

```java
public final View createView(String name, String prefix, AttributeSet attrs)
            throws ClassNotFoundException, InflateException {

        Constructor<? extends View> constructor = sConstructorMap.get(name);
        Class<? extends View> clazz = null;

        try {
            if (constructor == null) {

                // 缓存中不存在某View的构造方法，先new出来放缓存中
                clazz = mContext.getClassLoader().loadClass(
                        prefix != null ? (prefix + name) : name).asSubclass(View.class);
                
                ...

                constructor = clazz.getConstructor(mConstructorSignature);
                sConstructorMap.put(name, constructor);

            } else {
                ...
            }

            Object[] args = mConstructorArgs;
            args[1] = attrs;
            return constructor.newInstance(args);

        } catch (NoSuchMethodException e) {
            ...
        }
    }
```

这就是最后一步了，十分容易理解：
通过传进来的全类名，调用`newInstance`来创建一个这个类的实例并返回，返回的这个实例就是我们需要的View了。

结合上面的递归解析过程，每个层级的节点都会被生成一个个的View，并根据View的层级关系add到对应的直接父View（上层节点）中，最终返回一个包含了所有解析好的子View的布局根View。

至此，通过xml来加载View的整个原理就分析完成了。

## 总结

最后，我们再次回顾下上面的分析结果：

### inflate方法的参数关系
- `root != null， attachToRoot == true`
传进来的布局会被加载成为一个View并作为子View添加到root中，最终返回root；
而且这个布局根节点的android:layout_xxx参数会被解析用来设置View的大小；

- `root == null， attachToRoot无意义`
当root为空时，attachToRoot无论是什么都没有意义。此时传进来的布局会被加载成为一个View并直接返回；
布局根View的`android:layout_xxx`属性会被忽略，**即android:layout_xx属性只有依附在某个ViewGroup中才能生效**；

- `root != null， attachToRoot == false`
传进来的布局会被加载成为一个View并直接返回。
布局根View的`android:layout_xxx`属性会被解析成LayoutParams并设置在View上，此时root只用于设置布局根View的大小和位置。

### 加载xml布局的原理
其实就是从根节点开始，递归解析xml的每个节点，每一步递归的过程是：通过节点名称（全类名），使用ClassLoader创建对应类的实例，也就是View，然后，将这个View添加到它的上层节点（父View）。并同时会解析对应xml节点的属性作为View的属性。每个层级的节点都会被生成一个个的View，并根据View的层级关系add到对应的直接父View（上层节点）中，最终返回一个包含了所有解析好的子View的布局根View。






