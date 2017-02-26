---
title: Android setContentView源码解析
date: 2017-02-23 13:44:17
tags:
- android
- 事件传递
- 源码解析
categories: 
- android
- 源码解析
---


前置知识：Window和WindowManager

在Activity初始化的过程中，会调用Activity的attach方法，在该方法中会创建一个PhoneWindow的实例，将其作为Activity的mWindow成员变量。

在执行完了Activity#attach()方法之后，会执行Activity#onCreate()方法。

我们在Activity#onCreate()方法中会就调用setContentView()方法，我们将一个Layout的资源ID传入该方法，调用了该方法之后就将layout资源转换成ViewGroup了，之后就可以调用findViewById()查找ViewGroup中的各种View。

![](https://ww2.sinaimg.cn/large/006tNbRwgy1fczcvg0sbnj30dw099di1.jpg)

<!-- more -->

> 出处： *[Allen's Zone](http://allenfeng.com/)*
> 作者： *Allen Feng*

```java
public class MainActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setContentView(R.layout.activity_main);
        ...
    }
}
```

从setContentView开始:

```java
public class Activity extends ContextThemeWrapper
        implements Window.Callback ... {

    private Window mWindow;
    
    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window) {

        attachBaseContext(context);

        ...

        mWindow = new PhoneWindow(this, window);
        mWindow.setCallback(this);
        ...

    }

    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }

    public Window getWindow() {
        return mWindow;
    }
}
```

可以看到，Activity内部有一个Window类型的成员变量，而这个mWindow其实是PhoneWindow的实例。
setContentView有三个不同的重载方法，我们先从最常用的`setContentView(@LayoutRes int layoutResID)`开始分析，另外两个暂且放在一旁.

首先，它内部调用了PhoneWindow的setContentView：

```java
public class PhoneWindow extends Window implements MenuBuilder.Callback {

    ...

    // This is the view in which the window contents are placed. It is either
    // mDecor itself, or a child of mDecor where the contents go.
    private ViewGroup mContentParent;

    // This is the top-level view of the window, containing the window decor.
    private DecorView mDecor;

    @Override
    public void setContentView(int layoutResID) {
        if (mContentParent == null) {
            installDecor();
        } else {
            mContentParent.removeAllViews();
        }
        mLayoutInflater.inflate(layoutResID, mContentParent);
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
}
```

首先会判断PhoneWindow内部mContentParent成员变量是不是null，如果是会调用installDecor()方法，否则会先调用`removeAllViews()`移除所有的子View，从这里可以知道，setContentView是可以多次调用的。每次重新调用都会先清空原有的内容。

紧接着是使用LayoutInflater.inflate解析我们传进来的布局文件，并将解析出来的View做为一个子View放到mContentParent中。

下面一步是通过getCallback方法，通过Activity的源码可以看出，Activity实现了Window.Callback接口。Window.Callback包含了一系列有用的回调，如下：

```java
  public interface Callback {
       
        public boolean dispatchKeyEvent(KeyEvent event);
        
        public boolean dispatchKeyShortcutEvent(KeyEvent event);

        public boolean dispatchTouchEvent(MotionEvent event);
        
        public boolean dispatchTrackballEvent(MotionEvent event);

        public boolean dispatchGenericMotionEvent(MotionEvent event);

        public boolean dispatchPopulateAccessibilityEvent(AccessibilityEvent event);

        ...

        public void onContentChanged();

        ...
    }
```

看到`dispatchKeyEvent`是不是觉得很眼熟？没错，Activity正是实现了这个接口，通过dispatchKeyEvent方法，在Window接收到触摸事件时接收回调，开始实现事件分发的，关于触摸事件分发机制，可以查看[Android事件分发机制源码解析](http://allenfeng.com/2017/02/22/android-touch-event-transfer-mechanism/)这篇文章。

由于getCallback()返回的就是Activity本身，所以当mContentParent子View有变化时，会回调至Activity的onContentChanged()方法。




现在我们看看installDecor做了什么事情：
```java
private void installDecor() {
        if (mDecor == null) {
            mDecor = generateDecor();
            ...
        }

        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);

            mTitleView = (TextView)findViewById(com.android.internal.R.id.title);
            // 设置title和ActionBar的代码
            ...
        }
}

protected DecorView generateDecor() {
    return new DecorView(getContext(), -1);
} 

private final class DecorView extends FrameLayout implements RootViewSurfaceTaker {
    ...
}
```

首先判断mDecor是否为空，如果为空的话直接调用generateDecor创建一个DecorView对象出来，DecorView其实是FrameLayout的子类。
然后是调用generateLayout来初始化mContentParent：

```java
protected ViewGroup generateLayout(DecorView decor) {

        TypedArray a = getWindowStyle();

        ...
        // 解析并设置我们为Activity设置theme，如隐藏标题栏等

        if (a.getBoolean(com.android.internal.R.styleable.Window_windowNoTitle, false)) {
            requestFeature(FEATURE_NO_TITLE);
        } else if (a.getBoolean(com.android.internal.R.styleable.Window_windowActionBar, false)) {
            // Don't allow an action bar if there is no title.
            requestFeature(FEATURE_ACTION_BAR);
        }

        ...

        // Inflate the window decor.

        int layoutResource;
        int features = getLocalFeatures();

        // 根据设置的features为layoutResource赋上不同的布局文件id
        if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        com.android.internal.R.attr.dialogTitleIconsDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = com.android.internal.R.layout.screen_title_icons;
            }
            // XXX Remove this once action bar supports these features.
            removeFeature(FEATURE_ACTION_BAR);
            // System.out.println("Title Icons!");
        } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
                && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
            ...
        } else {
            // 默认布局文件
            layoutResource = com.android.internal.R.layout.screen_simple;
        }

        ...

        mDecor.startChanging();

        // 根据上一步拿到的layout id解析出一个View，并添加到mDecorView中
        View in = mLayoutInflater.inflate(layoutResource, null);
        decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));

        // 返回id为content的ViewGroup
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }

        if ((features & (1 << FEATURE_INDETERMINATE_PROGRESS)) != 0) {
            ProgressBar progress = getCircularProgressBar(false);
            if (progress != null) {
                progress.setIndeterminate(true);
            }
        }


        // 剩下的设置是对title和背景的设置
        if (getContainer() == null) {
            Drawable drawable = mBackgroundDrawable;
            if (mBackgroundResource != 0) {
                drawable = getContext().getResources().getDrawable(mBackgroundResource);
            }

            mDecor.setWindowBackground(drawable);
            
            ...
            ...
        }

        return contentParent;
    }
```

可以看到，这段代码主要做了三件事情：

1. 解析我们通过布局样式文件或者代码给Activity设置的各种主题，如显示、隐藏状态栏，设置字体颜色等，其实是对mLocalFeatures的值做各种设置


2. 根据上面设置好的mLocalFeatures，选择不同的根布局文件，如果没有任何额外设置的话，会使用`com.android.internal.R.layout.screen_simple`这个布局文件：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">

    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />

    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

根布局是一个LinearLayout，包含一个ViewStub，用于ActionBar的懒加载，以及一个id为**content**的FrameLayout

3. 根据上一步拿到的layout id解析出一个View，并作为子View添加到mDecorView中。
紧接着通过findViewById找到id为**content**的ViewGroup，即上面那个布局文件里的FrameLayout，这个FrameLayout就是我们准备返回的contentParent

4. 设置mDecorView的背景以及title内容和颜色，最后返回contentParent。




现在让我们回到PhoneWindow的setContentView方法，总结下上面的内容。

```java
public class PhoneWindow extends Window implements MenuBuilder.Callback {

    ...

    // 我们通过setContentView设置的布局的父View
    private ViewGroup mContentParent;

    // This is the top-level view of the window, containing the window decor.
    private DecorView mDecor;

    @Override
    public void setContentView(int layoutResID) {
        if (mContentParent == null) {
            installDecor();
        } else {
            mContentParent.removeAllViews();
        }
        mLayoutInflater.inflate(layoutResID, mContentParent);
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
}
```

通过上面的分析，我们知道installDecor首先创建了一个新的DecorView，然后根据设置选择不同的根布局xml文件，并解析出来这个布局文件作为一个子View添加到mDecor中，
最后返回id为**content**的mContentParent。

结合这个分析结果，现在来看上面`setContentView`剩下的内容，我们传进来的布局最终被解析出来，并添加到mContentParent中。也就是我们的执行完setContentView后，布局层次结构是这样子的：


我们通过一个简单的实例去验证下：

MainActivity.java
```java
public class MainActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```

activity_main.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:id="@+id/demo_rl"
    android:layout_height="match_parent">

    <TextView
        android:id="@+id/demo_tv"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerInParent="true"
        android:text="Hello World, Allen Feng !" />

</RelativeLayout>

```

为了减少干扰，我们将ActionBar隐藏：

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.feng.anr">

    <application
        ....
        android:theme="@android:style/Theme.Light.NoTitleBar">
        ....
    </application>

</manifest>
```

运行起来如图所示：

![运行结果](https://ww1.sinaimg.cn/large/006tNbRwgy1fd1e7qamh8j30u01hcdgp.jpg)


接着用我们熟悉的`Hierarchy View`查看下布局层级：

![布局层级](https://ww3.sinaimg.cn/large/006tNbRwgy1fd1ehcvd2mj31e20jk7bs.jpg)















