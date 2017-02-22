# Notes while reading Android source code

@(Android)[android|notes]

![](http://ww4.sinaimg.cn/large/65e4f1e6gw1f9g4mq1a2nj20hs0hs790.jpg)

[TOC]

## JNI

更为详细的资料：http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/jniTOC.html

### jni函数注册

#### 静态注册函数
- 先编写Java代码，然后编译生成`.class`文件（使用`javac`）。
- 使用Java的工具程序`javah`，如`javah –o output packagename.classname` ，这样它会生成一个叫output.h的JNI层头文件。其中packagename.classname是Java代码编译后的class文件，而在生成的output.h文件里，声明了对应的JNI层函数，只要实现里面的函数即可。


#### 动态注册函数
当Java层通过`System.loadLibrary`加载完JNI动态库后，紧接着会查找该库中一个叫`JNI_OnLoad`的函数，如果有，就调用它，而动态注册的工作就是在这里完成的。

```c++

jint JNI_OnLoad(JavaVM* vm, void* /* reserved */)
{
    JNIEnv* env = NULL;
    jint result = -1;

    if (vm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK) {
        ALOGE("ERROR: GetEnv failed\n");
        goto bail;
    }
    assert(env != NULL);

    if (register_android_media_MediaRecorder(env) < 0) {
        ALOGE("ERROR: MediaRecorder native registration failed\n");
        goto bail;
    }

    /* success -- return valid version number */
    result = JNI_VERSION_1_4;

bail:
    return result;
}

// 该函数用于注册Native函数，由JNI_OnLoad去调用
int register_android_media_MediaScanner(JNIEnv *env)
{
    return AndroidRuntime::registerNativeMethods(env,
                kClassMediaScanner, gMethods, NELEM(gMethods));
}

static const JNINativeMethod gMethods[] = {
    {
        "processDirectory”, // java中native函数的名字
        "(Ljava/lang/String;Landroid/media/MediaScannerClient;)V”, // Java函数的签名信息
        (void *)android_media_MediaScanner_processDirectory // JNI层对应的函数指针
    },
};
```

### 参数问题
```c++
static void android_media_MediaScanner_processFile( JNIEnv *env, jobject thiz, jstring path, jstring mimeType, jobject client)
```

- *JNIEnv *env* 字面意思是JNI的运行环境，最终指向的是一个名为JNINativeInterface的结构体，结构体里面定义了许多函数指针。可以这样理解，其实JNIEnv，就是对DVM运行环境中C、C++函数的一个引用，而也正因为此，当C、C++想要在DVM中调用函数的时候，由于其是在DVM的环境中，所以它们必须通过JNIEnv* 这个参数来获得这些方法，之后才能够使用。

- *jobject thiz* 第二个参数jobject代表Java层的MediaScanner对象，它表示是在哪个MediaScanner对象上调用的processFile方法。如果Java层是static函数的话，那么这个参数将是jclass，表示是在调用哪个Java Class的静态函数。

调用JavaVM的AttachCurrentThread函数，就可得到这个线程的JNIEnv结构体。这样就可以在后台线程中回调Java函数了

#### 如何通过JNIEnv操作jobject的成员函数
mEnv->CallVoidMethod(mClient, mScanFileMethodID, pathStr,
lastModified, fileSize);

#### 通过jfieldID操作jobject的成员变量
```c++
//获得fieldID后，可调用Get<type>Field系列函数获取jobject对应成员变量的值。

NativeType Get<type>Field(JNIEnv *env,jobject obj,jfieldID fieldID)

//或者调用Set<type>Field系列函数来设置jobject对应成员变量的值。

void Set<type>Field(JNIEnv *env,jobject obj,jfieldID fieldID,NativeType value)

//下面我们列出一些参加的Get/Set函数。

GetObjectField()         SetObjectField()

GetBooleanField()         SetBooleanField()

GetByteField()           SetByteField()

GetCharField()           SetCharField()

GetShortField()          SetShortField()

GetIntField()            SetIntField()

GetLongField()           SetLongField()


GetFloatField()          SetFloatField()
GetDoubleField()                  SetDoubleField()
```

### jstring的使用
·  调用JNIEnv的NewString(JNIEnv *env, const jchar*unicodeChars,jsize len)，可以从Native的字符串得到一个jstring对象。其实，可以把一个jstring对象看成是Java中String对象在JNI层的代表，也就是说，jstring就是一个Java String。但由于Java String存储的是Unicode字符串，所以NewString函数的参数也必须是Unicode字符串。


·  调用JNIEnv的NewStringUTF将根据Native的一个UTF-8字符串得到一个jstring对象。在实际工作中，这个函数用得最多。

demo:
```c++
[-->android_media_MediaScanner.cpp]

static void

android_media_MediaScanner_processFile(JNIEnv*env, jobject thiz, jstring path, jstring mimeType, jobject client)

{

   MediaScanner *mp = (MediaScanner *)env->GetIntField(thiz,fields.context);

......

//调用JNIEnv的GetStringUTFChars得到本地字符串pathStr

    constchar *pathStr = env->GetStringUTFChars(path, NULL);

......

//使用完后，必须调用ReleaseStringUTFChars释放资源

   env->ReleaseStringUTFChars(path, pathStr);


    ......
}
```

### 自动生成函数签名信息

![自动生成函数签名信息](http://ww4.sinaimg.cn/large/65e4f1e6gw1f9g4qrwiu8j20qs0e8q6i.jpg)

### JNI的三种引用
- Local Reference
- Global Reference
- Weak Global Reference

### 异常处理：
JNI中的异常不会中断本地函数的执行 会在JNI返回到JAVA层之后 虚拟机才会抛出这个异常

“ExceptionOccured函数，用来判断是否发生异常。

ExceptionClear函数，用来清理当前JNI层中发生的异常。

ThrowNew函数，用来向Java层抛出异常。”


## init工作流程

可将init的工作流程精简为以下四点：
- 解析两个配置文件，其中，将分析对init.rc文件的解析。
- 执行各个阶段的动作，创建Zygote的工作就是在其中的某个阶段完成的。
- 调用`property_init`初始化属性相关的资源，并且通过`property_start_service`启动属性服务。
- init进入一个无限循环，并且等待一些事情的发生。重点关注init如何处理来自socket和来自属性服务器相关的事情。


## Zygote创建Java世界的步骤
1. 创建虚拟机
startVm 
设置一系列参数
包括虚拟机的heapSize等

2. 注册JNI函数
startReg
给后续java世界调用的一些函数提前注册

3. CallStaticVoidMethod 调用com.android.internal.os.ZygoteInit的main函数
开始进入Java世界
(1) 建立IPC通信服务端-registerZygoteSocket: 供zygote与系统中其他程序的通信
(2) 预加载类（约一千多个）和系统资源(framework-res.apk)
(3) 启动system_server(fork自zygote)
(4) 等待处理客户连接和客户请求



Zygote响应请求的过程：

startActivity时，由SystemService进程中的ActivityManagerService向zygote进程发送请求，fork一个子进程，子进程调用android.app.ActivityThread的main函数，完成app启动
zygote 预加载

## WatchDog

## Android的指针管理： RefBase、SP和WP
Android中通过引用计数来实现智能指针，并且实现有强指针与弱指针。由对象本身来提供引用计数器，但是对象不会去维护引用计数器的值，而是由智能指针来管理。

### 轻量级引用计数的实现：LightRefBase
LightRefBase的实现很简单，只是内部保存了一个变量用于保存对象被引用的次数，并提供了两个函数用于增加或减少引用计数。

### sp(Strong Pointer)
LightRefBase仅仅提供了引用计数的方法，具体引用数应该怎么管理，就要通过智能指针类来管理了，每当有一个智能指针指向对象时，对象的引用计数要加1，当一个智能指针取消指向对象时，对象的引用计数要减1。
在C＋＋中，当一个对象生成和销毁时会自动调用（拷贝）构造函数和析构函数，所以，对对象引用数的管理就可以放到智能指针的（拷贝）构造函数和析构函数中。Android提供了一个智能指针可以配合LightRefBase使用：sp

SP的构造函数做了如下事情：
默认的构造函数使智能指针不指向任何对象。
内部变量m_ptr指向实际对象，并调用实际对象的incStrong函数，T继承自LightRefBase，所以此处调用的是LightRefBase的incStrong函数，之后实际对象的引用计数加1。
```c++
template<typename T>
sp<T>::sp(const sp<T>& other)
: m_ptr(other.m_ptr)
{
    if (m_ptr) m_ptr->incStrong(this);
}
```


当智能指针销毁的时候调用智能指针的析构函数：
```c++
template<typename T>
sp<T>::~sp()
{
    if (m_ptr) m_ptr->decStrong(this);
}
```

调用实际对象即LightRefBase的decStrong函数，其实现如下：
```c++
inline void decStrong(const void* id) const {
    if (android_atomic_dec(&mCount) == 1) {
        delete static_cast<const T*>(this);
    }
}
```

android_atomic_dec返回mCount减1之前的值，如果返回1表示这次减过之后引用计数就是0了，就把对象delete掉。


> 首先所有的类都会虚继承refbase类，因为它实现了达到垃圾回收所需要的所有function，因此实际上所有的对象声明出来以后都具备了自动 释放自己的能力，也就是说实际上智能指针就是我们的对象本身，它会维持一个对本身强引用和弱引用的计数，一旦强引用计数为0它就会释放掉自己。 

## Thread和常用同步类分析

### Threads类
若mCanCallJava为true，它创建的新线程将：
- 在调用你的线程函数之前会attach到 JNI环境中，这样，你的线程函数就可以无忧无虑地使用JNI函数了。
- 线程函数退出后，它会从JNI环境中detach，释放一些资源。

第二点尤其重要，因为进程退出前，dalvik虚拟机会检查是否有attach了，但是最后未detach的线程如果有，则会直接abort（这不是一件好事）。这儿是为了Dalvik的健康而言。

但是，不论是否存在上述参数，最终的线程函数_threadLoop都会被调用。

threadLoop运行在一个循环中，它的返回值可以决定是否退出线程。

### 互斥类-Mutex
用于线程同步，关于Mutex的使用，除了初始化外，最重要的是lock和unlock函数的使用，它们的用法如下：

·  要想独占卫生间，必须先调用Mutex的lock函数。这样，这个区域就被锁住了。如果这块区域之前已被别人锁住，lock函数则会等待，直到可以进入这块区域为止。系统保证一次只有一个线程能lock成功。

·  当你“方便”完毕，记得调用Mutex的unlock以释放互斥区域。这样，其他人的lock才可以成功返回。

·  另外，Mutex还提供了一个trylock函数，该函数只是尝试去锁住该区域，使用者需要根据trylock的返回值判断是否成功锁住了该区域。

使用：
·  显式调用Mutex的lock。
·  在某个时候要记住调用该Mutex的unlock。
以上这些操作都必须一一对应，否则会出现“死锁”！

### AutoLock-自动锁
基于Mutex，但是使用简单的多，主要是利用构造函数和析构函数的特性分别锁住和解锁。

AutoLock的用法很简单：

·  先定义一个Mutex，如 Mutex xlock；
·  在使用xlock的地方，定义一个AutoLock，如 AutoLock autoLock（xlock）。

聪明！

### 什么是原子操作
该操作绝不会在执行完毕前被任何其他任务或事件打断，也就说，原子操作是最小的执行单位。


## Android日志
c++层打印日志：

include system/core/include/cutils/log.h 即可

```c++
#define LOG_TAG "MY LOG TAG"
#include <cutils/log.h>

void main() {
    LOGV("This is the log printed by LOGV in android user space.");    
}
```


java层日志
frameworks/base/core/java/android/util/Log.java

交由native层打印日志：
```java
................................................  
  
public final class Log {  
  
................................................  
  
    /** 
     * Priority constant for the println method; use Log.v. 
         */  
    public static final int VERBOSE = 2;  
  
    /** 
     * Priority constant for the println method; use Log.d. 
         */  
    public static final int DEBUG = 3;  
  
    /** 
     * Priority constant for the println method; use Log.i. 
         */  
    public static final int INFO = 4;  
  
    /** 
     * Priority constant for the println method; use Log.w. 
         */  
    public static final int WARN = 5;  
  
    /** 
     * Priority constant for the println method; use Log.e. 
         */  
    public static final int ERROR = 6;  
  
    /** 
     * Priority constant for the println method. 
         */  
    public static final int ASSERT = 7;  
  
.....................................................  
  
    public static int v(String tag, String msg) {  
        return println_native(LOG_ID_MAIN, VERBOSE, tag, msg);  
    }  
  
    public static int v(String tag, String msg, Throwable tr) {  
        return println_native(LOG_ID_MAIN, VERBOSE, tag, msg + '\n' + getStackTraceString(tr));  
    }  
  
    public static int d(String tag, String msg) {  
        return println_native(LOG_ID_MAIN, DEBUG, tag, msg);  
    }  
  
    public static int d(String tag, String msg, Throwable tr) {  
        return println_native(LOG_ID_MAIN, DEBUG, tag, msg + '\n' + getStackTraceString(tr));  
    }  
  
    public static int i(String tag, String msg) {  
        return println_native(LOG_ID_MAIN, INFO, tag, msg);  
    }  
  
    public static int i(String tag, String msg, Throwable tr) {  
        return println_native(LOG_ID_MAIN, INFO, tag, msg + '\n' + getStackTraceString(tr));  
    }  
  
    public static int w(String tag, String msg) {  
        return println_native(LOG_ID_MAIN, WARN, tag, msg);  
    }  
  
    public static int w(String tag, String msg, Throwable tr) {  
        return println_native(LOG_ID_MAIN, WARN, tag, msg + '\n' + getStackTraceString(tr));  
    }  
  
    public static int w(String tag, Throwable tr) {  
        return println_native(LOG_ID_MAIN, WARN, tag, getStackTraceString(tr));  
    }  
      
    public static int e(String tag, String msg) {  
        return println_native(LOG_ID_MAIN, ERROR, tag, msg);  
    }  
  
    public static int e(String tag, String msg, Throwable tr) {  
        return println_native(LOG_ID_MAIN, ERROR, tag, msg + '\n' + getStackTraceString(tr));  
    }  
  
..................................................................  
  
    /**@hide */ public static native int println_native(int bufID,  
        int priority, String tag, String msg);  
}  
```

使用：
```java
private static final String LOG_TAG = "MY_LOG_TAG";
Log.i(LOG_TAG, "This is the log printed by Log.i in android user space.");
```

*/Users/fengjun/Developement/android-source-code/frameworks/base/core/jni/android_util_Log.cpp*


Android设备日志路径：

```c++
#define LOGGER_LOG_RADIO     "log_radio"     /* radio-related messages */
#define LOGGER_LOG_EVENTS    "log_events"    /* system/hardware events */
#define LOGGER_LOG_MAIN      "log_main"      /* everything else */
```

```bash
root@NX529J:/dev/log # pwd
/dev/log
root@NX529J:/dev/log # ls
events
main
radio
system
root@NX529J:/dev/log #
```



```c++
#define LOG_NAMESPACE "log.tag."
#define LOG_TAG "Log_println"

#include <assert.h>
#include <cutils/properties.h>
#include <utils/Log.h>
#include <utils/String8.h>

#include "jni.h"
#include "utils/misc.h"
#include "android_runtime/AndroidRuntime.h"

#define MIN(a,b) ((a<b)?a:b)

namespace android {

struct levels_t {
    jint verbose;
    jint debug;
    jint info;
    jint warn;
    jint error;
    jint assert;
};
static levels_t levels;

static int toLevel(const char* value) 
{
    switch (value[0]) {
        case 'V': return levels.verbose;
        case 'D': return levels.debug;
        case 'I': return levels.info;
        case 'W': return levels.warn;
        case 'E': return levels.error;
        case 'A': return levels.assert;
        case 'S': return -1; // SUPPRESS
    }
    return levels.info;
}

static jboolean android_util_Log_isLoggable(JNIEnv* env, jobject clazz, jstring tag, jint level)
{
#ifndef HAVE_ANDROID_OS
    return false;
#else /* HAVE_ANDROID_OS */
    int len;
    char key[PROPERTY_KEY_MAX];
    char buf[PROPERTY_VALUE_MAX];

    if (tag == NULL) {
        return false;
    }
    
    jboolean result = false;
    
    const char* chars = env->GetStringUTFChars(tag, NULL);

    if ((strlen(chars)+sizeof(LOG_NAMESPACE)) > PROPERTY_KEY_MAX) {
        jclass clazz = env->FindClass("java/lang/IllegalArgumentException");
        char buf2[200];
        snprintf(buf2, sizeof(buf2), "Log tag \"%s\" exceeds limit of %d characters\n",
                chars, PROPERTY_KEY_MAX - sizeof(LOG_NAMESPACE));

        // release the chars!
        env->ReleaseStringUTFChars(tag, chars);

        env->ThrowNew(clazz, buf2);
        return false;
    } else {
        strncpy(key, LOG_NAMESPACE, sizeof(LOG_NAMESPACE)-1);
        strcpy(key + sizeof(LOG_NAMESPACE) - 1, chars);
    }
    
    env->ReleaseStringUTFChars(tag, chars);

    len = property_get(key, buf, "");
    int logLevel = toLevel(buf);
    return (logLevel >= 0 && level >= logLevel) ? true : false;
#endif /* HAVE_ANDROID_OS */
}

/*
 * In class android.util.Log:
 *  public static native int println_native(int buffer, int priority, String tag, String msg)
 */
static jint android_util_Log_println_native(JNIEnv* env, jobject clazz,
        jint bufID, jint priority, jstring tagObj, jstring msgObj)
{
    const char* tag = NULL;
    const char* msg = NULL;

    if (msgObj == NULL) {
        jclass npeClazz;

        npeClazz = env->FindClass("java/lang/NullPointerException");
        assert(npeClazz != NULL);

        env->ThrowNew(npeClazz, "println needs a message");
        return -1;
    }

    if (bufID < 0 || bufID >= LOG_ID_MAX) {
        jclass npeClazz;

        npeClazz = env->FindClass("java/lang/NullPointerException");
        assert(npeClazz != NULL);

        env->ThrowNew(npeClazz, "bad bufID");
        return -1;
    }

    if (tagObj != NULL)
        tag = env->GetStringUTFChars(tagObj, NULL);
    msg = env->GetStringUTFChars(msgObj, NULL);

    int res = __android_log_buf_write(bufID, (android_LogPriority)priority, tag, msg);

    if (tag != NULL)
        env->ReleaseStringUTFChars(tagObj, tag);
    env->ReleaseStringUTFChars(msgObj, msg);

    return res;
}

/*
 * JNI registration.
 */
static JNINativeMethod gMethods[] = {
    /* name, signature, funcPtr */
    { "isLoggable",      "(Ljava/lang/String;I)Z", (void*) android_util_Log_isLoggable },
    { "println_native",  "(IILjava/lang/String;Ljava/lang/String;)I", (void*) android_util_Log_println_native },
};

int register_android_util_Log(JNIEnv* env)
{
    jclass clazz = env->FindClass("android/util/Log");

    if (clazz == NULL) {
        LOGE("Can't find android/util/Log");
        return -1;
    }
    
    levels.verbose = env->GetStaticIntField(clazz, env->GetStaticFieldID(clazz, "VERBOSE", "I"));
    levels.debug = env->GetStaticIntField(clazz, env->GetStaticFieldID(clazz, "DEBUG", "I"));
    levels.info = env->GetStaticIntField(clazz, env->GetStaticFieldID(clazz, "INFO", "I"));
    levels.warn = env->GetStaticIntField(clazz, env->GetStaticFieldID(clazz, "WARN", "I"));
    levels.error = env->GetStaticIntField(clazz, env->GetStaticFieldID(clazz, "ERROR", "I"));
    levels.assert = env->GetStaticIntField(clazz, env->GetStaticFieldID(clazz, "ASSERT", "I"));
                
    return AndroidRuntime::registerNativeMethods(env, "android/util/Log", gMethods, NELEM(gMethods));
}

}; // namespace android


```


至些，整个调用过程就结束了。总结一下，首先是从应用程序层调用应用程序框架层的Java接口，应用程序框架层的Java接口通过调用本层的JNI方法进入到系统运行库层的C接口，系统运行库层的C接口通过设备文件来访问内核空间层的Logger驱动程序。


## Binder
Binder是Android系统提供一种IPC（进程间通讯）机制。(其他可采用Socket或Pipe等)
Android系统基本上可以看作是一个基于Binder通信的C/S架构。

这个C/S结构不同于以往的普通结构：
![](http://ww3.sinaimg.cn/large/65e4f1e6gw1f9rwgik18ej20am04m74e.jpg)

在基于Binder通信的C/S架构体系中，除了C/S架构所包括的Client端和Server端外，Android还有一个全局的ServiceManager端，它的作用是管理系统中的各种服务（Service）。

- Server进程要先注册一些Service到ServiceManager中，所以Server是ServiceManager的客户端，而ServiceManager就是服务端了。
- 如果某个Client进程要使用某个Service，必须先到ServiceManager中获取该Service的相关信息，所以Client是ServiceManager的客户端。
- Client根据得到的Service信息建立与Service所在的Server进程通信的通路，然后就可以直接与Service交互了，所以Client也是Server的客户端。


Android基于Linux内核，Linux内核原有的通信方式：
传统的管道（Pipe）、信号（Signal）和跟踪（Trace）
报文队列（Message）、共享内存（Share Memory）和信号量（Semaphore）；后来BSD Unix对“System V IPC”机制进行了重要的扩充，提供了一种称为插口（Socket）的进程间通信机制。


![](http://hi.csdn.net/attachment/201107/19/0_13110996490rZN.gif)

1. Client、Server和Service Manager实现在用户空间中，Binder驱动程序实现在内核空间中

2. Binder驱动程序和Service Manager在Android平台中已经实现，开发者只需要在用户空间实现自己的Client和Server

3. Binder驱动程序提供设备文件/dev/binder与用户空间交互，Client、Server和Service Manager通过open和ioctl文件操作函数与Binder驱动程序进行通信

4. Client和Server之间的进程间通信通过Binder驱动程序间接实现

5. Service Manager是一个守护进程，用来管理Server，并向Client提供查询Server接口的能力


# Analysis on Android source code

## 智能指针
c++写代码最容易出问题的地方就是指针，具体表现为忘记释放指针指向的对象所占用的内存。
轻则内存泄露，重则造成莫名其妙的逻辑错误，甚至系统崩溃。
出于性能考虑，有时候还是不得不使用c++代码。
因此Android系统为我们提供了C++智能指针，避免出现指针使用不当的问题。

常用对象对象引用计数来维护对象生命周期。

智能指针是一种能够自动维护对象引用的技术，智能指针是一个对象，而不是一个指针。

Android智能指针分为以下三种：
- 轻量级指针
- 强指针
- 弱指针


## Logger日志系统
Logger日志系统基于内核中的Logger日志驱动程序，它将日志保存在一个内存空间中。
使用唤醒缓冲区，所以日志满了以后，新的日志将会覆盖旧的日志。

四个设备文件：
`/dev/log/main` 应用程序级 `android.util.Log`
`/dev/log/system` 系统 `android.util.Slog`
`/dev/log/radio` 无线设备相关  
`/dev/log/events` 专用于诊断系统问题 `android.util.EventLog`

## Binder进程间通讯系统
Linux原有进程间通讯机制：
- 管道 pipe
- 信号 signal
- 消息队列 message
- 共享内存 share memory
- 插口 sockect

Binder只需要进行*一次拷贝操作*，节省空间和效率

C/S通信机制

提供服务的叫Server进程
访问服务的叫Client进程

每一个Server进程和Client进程都维护一个Binder线程池来处理进程间的通信请求
因此Server进程和Client进程可以并发地提供和访问服务

通信有Binder驱动程序进行，Binder驱动程序向用户空间暴露了一个设备文件/dev/binder


![](http://ww3.sinaimg.cn/large/006y8lVagw1fafqcmijq0j31940ry795.jpg)


四个场景：
1. Service Manager的启动过程
2. Service Manger代理对象的获取过程
3. Service组件的启动过程
4. Service代理对象的获取过程






## Ashmem匿名共享内存系统

## Activity组件的启动过程
点击桌面Launcher图标后，启动一个Activity组件的过程：

1. Launcher组件向ActivityManagerService发送了一个启动MainActivity组件的进程间通信请求
2. ActivityManagerService将要启动的MainActivity组件的信息保存下来，然后先Launcher组件发送一个进入中止状态的进程间通信请求。
3. Launcher组件进入到中止状态后。就会向ActivityManagerService发送一个已进入终止状态的进程间通信请求，以便ActivityManagerService可以继续启动MainActivity组件的操作
4. ActivityManagerService发现用来运行Activity的应用程序进程不存在，因此它会先启动一个新的应用程序进程。
5. 新的应用程序进程启动完成以后，回想ActivityManagerService发送一个启动完成的进程间通信请求，以便ActivityManagerService可以继续执行启动MainActivity组件的操作
6. ActivityManagerService将第二步保存下来的MainActivity组件的信息发送给第四步创建的应用程序进程，以便它可以将MainActivity组件启动起来






过程解析：

先从点击事件下手：

```java
/**
 * Launches the intent referred by the clicked shortcut.
 *
 * @param v The view representing the clicked shortcut.
 */
public void onClick(View v) {
    Object tag = v.getTag();
    if (tag instanceof ShortcutInfo) {
        // Open shortcut
        final Intent intent = ((ShortcutInfo) tag).intent;
        int[] pos = new int[2];
        v.getLocationOnScreen(pos);
        intent.setSourceBounds(new Rect(pos[0], pos[1],
                pos[0] + v.getWidth(), pos[1] + v.getHeight()));
        startActivitySafely(intent, tag);
    } else if (tag instanceof FolderInfo) {
        handleFolderClick((FolderInfo) tag);
    } else if (v == mHandleView) {
        if (isAllAppsVisible()) {
            closeAllApps(true);
        } else {
            showAllApps(true);
        }
    }
}
```

只需关注第一个if即可，可以看到点击缩略图表后，会通过startActivitySafely来启动Activity，需要启动的Activity的信息被包含在intent中:

这个Intent是在Lauchner启动的时候，通过PackageManagerService获取所有已安装应用中，在AndroidManifest.xml中包含以下IntentFilter的Activity列表：

```xml
<intent-filter>
    <action android:name="android.intent.action.MAIN" />
    <category android:name="android.intent.category.LAUNCHER" />
</intent-filter>
```

Intent的load过程如下

```java
private void loadAllAppsByBatch() {

    ....

    final Intent mainIntent = new Intent(Intent.ACTION_MAIN, null);
    mainIntent.addCategory(Intent.CATEGORY_LAUNCHER);

    final PackageManager packageManager = mContext.getPackageManager();
    List<ResolveInfo> apps = null;

    ...

    while (i < N && !mStopped) {
        synchronized (mAllAppsListLock) {
            if (i == 0) {
                
                mAllAppsList.clear();
                final long qiaTime = DEBUG_LOADERS ? SystemClock.uptimeMillis() : 0;
                apps = packageManager.queryIntentActivities(mainIntent, 0);
                
                ...
            }

            final long t2 = DEBUG_LOADERS ? SystemClock.uptimeMillis() : 0;

            ...
        }

        ...
    }

    ...
}
 ```


现在回到我们的主线。startActivitySafely方法：
        
```java
void startActivitySafely(Intent intent, Object tag) {
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    try {
        startActivity(intent);
    } catch (ActivityNotFoundException e) {
        Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
        Log.e(TAG, "Unable to launch. tag=" + tag + " intent=" + intent, e);
    } catch (SecurityException e) {
        Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
        Log.e(TAG, "Launcher does not have the permission to launch " + intent +
                ". Make sure to create a MAIN intent-filter for the corresponding activity " +
                "or use the exported attribute for this activity. "
                + "tag="+ tag + " intent=" + intent, e);
    }
}
```

这个方法主要调用了startActivity(intent);来启动我们的Activity。
值得注意的是，Launcher会自动加上一个Intent.FLAG_ACTIVITY_NEW_TASK，以便于新的Activity可以在一个新的任务栈中启动，

关于这FLAG见：。。。


接着调用的startActivity，而startActivity最后调用到也是Activity类中的：

```java
public void startActivityForResult(Intent intent, int requestCode) {
        if (mParent == null) {
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode);

            ...

            }
        } else {
            mParent.startActivityFromChild(this, intent, requestCode);
        }
    }
```

Instrumentation是一个用来监控应用程序和系统之间操作的类。启动Activity的关键代码为mInstrumentation.execStartActivity，便于使用Instrumentation监控这个过程。

mMainThread类型为ActivityThread，在

```java
final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            Object lastNonConfigurationInstance,
            HashMap<String,Object> lastNonConfigurationChildInstances,
            Configuration config) {
        attachBaseContext(context);

        ...

        mMainThread = aThread;
}
```

















Context who, 
IBinder contextThread, 
IBinder token, 
Activity target,
Intent intent, 
int requestCode





```java
public ActivityResult execStartActivity(
    Context who, IBinder contextThread, IBinder token, Activity target,
    Intent intent, int requestCode) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    if (mActivityMonitors != null) {
        synchronized (mSync) {
            final int N = mActivityMonitors.size();
            for (int i=0; i<N; i++) {
                final ActivityMonitor am = mActivityMonitors.get(i);
                if (am.match(who, null, intent)) {
                    am.mHits++;
                    if (am.isBlocking()) {
                        return requestCode >= 0 ? am.getResult() : null;
                    }
                    break;
                }
            }
        }
    }
    try {
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    null, 0, token, target != null ? target.mEmbeddedID : null,
                    requestCode, false, false);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
    }
    return null;
}
```

















## Service组件的启动过程

## Android系统广播机制

## Content Provider组件的实现原理

## Zygote和System进程的启动过程

## Android应用程序进程的启动过程

## Android应用程序的消息处理机制

## Android应用程序的键盘消息处理机制

## Android应用程序线程的消息循环模型

## Android应用程序的安装和显示过程
































