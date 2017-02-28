---
title: Android应用启动流程解析
date: 2017-02-28 10:09:38
tags:
- android
- 启动
- 源码分析
---

应用启动速度优化也是我们需要关注的一大话题。要优化启动时间，第一步就是了解应用启动的工作原理。本文主要基于Android 6.0源码，梳理应用启动的整体流程。

阅读完本文，我们可以解答下列问题：
- 应用层可以接触到的最早方法是什么？
- 主要经历哪些步骤？
- 巴拉巴拉

## 从点击图标说起
用户启动一个应用程序的最常见方式是在桌面（启动器）点击对应的App图标，我们以此为起点开始追踪。启动器也是一个Activity，点击事件对应源码如下：

```java
public final class Launcher extends Activity
        implements View.OnClickListener ... {
        
	public void onClick(View v) {
	    
	    Object tag = v.getTag();
	    if (tag instanceof ShortcutInfo) {
	        ...
	        final Intent intent = ((ShortcutInfo) tag).intent;

	        ...
	        boolean success = startActivitySafely(intent, tag);

	        ...
	    } 
	    ...
	}

	boolean startActivitySafely(Intent intent, Object tag) {
	    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
	    try {
	        startActivity(intent);
	        return true;
	    } catch (ActivityNotFoundException e) {
	        ...
	    } catch (SecurityException e) {
	        ...
	    }
	    return false;
	}
}	
```

点击某个图标（View）后，根据这个View的tag字段获取到Intent，并调用startActivity启动应用的LAUNCHER Activity。startActivitySafely只是对startActivity加了try-catch保护。

## IPC调用至ActivityManagerService

startActivity对应实现在Activity中：

```java
public class Activity extends ContextThemeWrapper implements ... {

	@Override
	public void startActivity(Intent intent) {
	    startActivityForResult(intent, -1);
	}

	public void startActivityForResult(Intent intent, int requestCode) {
	    if (mParent == null) {
	        Instrumentation.ActivityResult ar =
	            mInstrumentation.execStartActivity(
	                this, mMainThread.getApplicationThread(), mToken, this,
	                intent, requestCode);
	        ...
	    } else {
	        ...
	    }
	}
}
```

Activity内部调用了Instrumentation的方法：

```java
public class Instrumentation {
    
    ...

	public ActivityResult execStartActivity(
	        Context who, IBinder contextThread, IBinder token, Activity target,
	        Intent intent, int requestCode) {
	    IApplicationThread whoThread = (IApplicationThread) contextThread;
	    
	    ...

	    try {
	        intent.setAllowFds(false);
	        int result = ActivityManagerNative.getDefault()
	            .startActivity(whoThread, intent,
	                    intent.resolveTypeIfNeeded(who.getContentResolver()),
	                    null, 0, token, target != null ? target.mEmbeddedID : null,
	                    requestCode, false, false, null, null, false);
	        checkStartActivityResult(result, intent);
	    } catch (RemoteException e) {
	    }
	    return null;
	}
}
```

由于AMS是系统核心服务，很多API不能开放供客户端使用，所以设计者没有让ActivityManager直接加入AMS家族。在ActivityManager类内部通过调用AMN的getDefault函数得到一个ActivityManagerProxy对象，通过它可与AMS通信。ActivityManagerNative.getDefault()获得的是一个ActivityManagerProxy实例。

```java
class ActivityManagerProxy implements IActivityManager
{   
    public int startActivity(IApplicationThread caller, Intent intent,
            String resolvedType, Uri[] grantedUriPermissions, int grantedMode,
            IBinder resultTo, String resultWho,
            int requestCode, boolean onlyIfNeeded,
            boolean debug, String profileFile, ParcelFileDescriptor profileFd,
            boolean autoStopProfiler) throws RemoteException {

        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        ...

        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        
        ...
        return result;
    }
}
```
ActivityManagerProxy通过Binder IPC机制发起了一个`START_ACTIVITY_TRANSACTION`命令，调用到ActivityManagerService，**接下来流程由Launcher所在进程进入了另一进程：system_server进程**继续执行。

SystemServer进程是在Android系统启动后，由Zygote孵化出来的，会启动所有系统核心服务, 例如Activity Manager Service, 硬件相关的Service等。

```java
public final class ActivityManagerService extends ActivityManagerNative {
	....

	@Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        try {
            return super.onTransact(code, data, reply, flags);
        } catch (RuntimeException e) {
           ...
        }
    }
}
```

###

ActivityManagerService为ActivityManagerNative实现类，其对远程调用的处理依旧在父类中：

```java
public abstract class ActivityManagerNative extends Binder implements IActivityManager
{
    ...

	public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
	            throws RemoteException {
	        switch (code) {
	        case START_ACTIVITY_TRANSACTION:
	        {

	            ...
	            int result = startActivity(app, intent, resolvedType,
	                    grantedUriPermissions, grantedMode, resultTo, resultWho,
	                    requestCode, onlyIfNeeded, debug, profileFile, profileFd, autoStopProfiler);
	            ...
	            return true;
	        }
	        ...
	        }
	        ...
	}
}
```

注意IActivityManager接口定义了startActivity等方法，实现在ActivityManagerService中：
```java
public final class ActivityManagerService extends ActivityManagerNative {
	
	...

	public final int startActivity(IApplicationThread caller,
	        Intent intent, String resolvedType, Uri[] grantedUriPermissions,
	        int grantedMode, IBinder resultTo,
	        String resultWho, int requestCode, boolean onlyIfNeeded, boolean debug,
	        String profileFile, ParcelFileDescriptor profileFd, boolean autoStopProfiler) {

	    return mMainStack.startActivityMayWait(caller, -1, intent, resolvedType,
	            grantedUriPermissions, grantedMode, resultTo, resultWho,
	            requestCode, onlyIfNeeded, debug, profileFile, profileFd, autoStopProfiler,
	            null, null);
	}
}
```

mMainStack为ActivityStack实例，对应Activity管理中的返回栈概念，详见[Tasks and Back Stack](https://developer.android.com/guide/components/activities/tasks-and-back-stack.html)。

```java
final class ActivityStack {

    ...

	final int startActivityMayWait(IApplicationThread caller, int callingUid,
	            Intent intent, String resolvedType, Uri[] grantedUriPermissions,
	            int grantedMode, IBinder resultTo,
	            String resultWho, int requestCode, boolean onlyIfNeeded,
	            boolean debug, String profileFile, ParcelFileDescriptor profileFd,
	            boolean autoStopProfiler, WaitResult outResult, Configuration config) {

	        ...

	        intent = new Intent(intent);

	        // 收集Intent指向的Activity信息
	        ActivityInfo aInfo = resolveActivity(intent, resolvedType, debug,
	                profileFile, profileFd, autoStopProfiler);

	        synchronized (mService) {
	            ...
	            
	            int res = startActivityLocked(caller, intent, resolvedType,
	                    grantedUriPermissions, grantedMode, aInfo,
	                    resultTo, resultWho, requestCode, callingPid, callingUid,
	                    onlyIfNeeded, componentSpecified, null);
	            
	            ...
	            
	            return res;
	        }
	 }

    ActivityInfo resolveActivity(Intent intent, 
    	String resolvedType, boolean debug,
        String profileFile, ParcelFileDescriptor profileFd, 
        boolean autoStopProfiler) {

	    ActivityInfo aInfo;
	    try {
	        ResolveInfo rInfo = AppGlobals.getPackageManager().resolveIntent(
	                    intent, resolvedType,
	                    PackageManager.MATCH_DEFAULT_ONLY
	                    | ActivityManagerService.STOCK_PM_FLAGS);
	        aInfo = rInfo != null ? rInfo.activityInfo : null;
	    } catch (RemoteException e) {
	        aInfo = null;
	    }
	    ...

	    return aInfo;
	}

 }
```

先调用resolveActivity，内部通过PackageManager的收集这个Intent对象的指向信息，并存在一个Intent对象中。

接着调用startActivityLocked：

```java
private final void startActivityLocked(ActivityRecord r, boolean newTask,
            boolean doResume, boolean keepCurTransition) {
        final int NH = mHistory.size();

        int addPos = -1;
        
        if (!newTask) {
            // If starting in an existing task, find where that is...
            boolean startIt = true;
            for (int i = NH-1; i >= 0; i--) {
                ActivityRecord p = mHistory.get(i);
                if (p.finishing) {
                    continue;
                }
                if (p.task == r.task) {
                    // Here it is!  Now, if this is not yet visible to the
                    // user, then just add it without starting; it will
                    // get started when the user navigates back to it.
                    addPos = i+1;
                    if (!startIt) {
                        if (DEBUG_ADD_REMOVE) {
                            RuntimeException here = new RuntimeException("here");
                            here.fillInStackTrace();
                            Slog.i(TAG, "Adding activity " + r + " to stack at " + addPos,
                                    here);
                        }
                        mHistory.add(addPos, r);
                        r.putInHistory();
                        mService.mWindowManager.addAppToken(addPos, r, r.task.taskId,
                                r.info.screenOrientation, r.fullscreen);
                        if (VALIDATE_TOKENS) {
                            mService.mWindowManager.validateAppTokens(mHistory);
                        }
                        return;
                    }
                    break;
                }
                if (p.fullscreen) {
                    startIt = false;
                }
            }
        }

        // Place a new activity at top of stack, so it is next to interact
        // with the user.
        if (addPos < 0) {
            addPos = NH;
        }
        
        // If we are not placing the new activity frontmost, we do not want
        // to deliver the onUserLeaving callback to the actual frontmost
        // activity
        if (addPos < NH) {
            mUserLeaving = false;
            if (DEBUG_USER_LEAVING) Slog.v(TAG, "startActivity() behind front, mUserLeaving=false");
        }
        
        // Slot the activity into the history stack and proceed
        if (DEBUG_ADD_REMOVE) {
            RuntimeException here = new RuntimeException("here");
            here.fillInStackTrace();
            Slog.i(TAG, "Adding activity " + r + " to stack at " + addPos, here);
        }
        mHistory.add(addPos, r);
        r.putInHistory();
        r.frontOfTask = newTask;
        if (NH > 0) {
            // We want to show the starting preview window if we are
            // switching to a new task, or the next activity's process is
            // not currently running.
            boolean showStartingIcon = newTask;
            ProcessRecord proc = r.app;
            if (proc == null) {
                proc = mService.mProcessNames.get(r.processName, r.info.applicationInfo.uid);
            }
            if (proc == null || proc.thread == null) {
                showStartingIcon = true;
            }
            if (DEBUG_TRANSITION) Slog.v(TAG,
                    "Prepare open transition: starting " + r);
            if ((r.intent.getFlags()&Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {
                mService.mWindowManager.prepareAppTransition(
                        WindowManagerPolicy.TRANSIT_NONE, keepCurTransition);
                mNoAnimActivities.add(r);
            } else if ((r.intent.getFlags()&Intent.FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET) != 0) {
                mService.mWindowManager.prepareAppTransition(
                        WindowManagerPolicy.TRANSIT_TASK_OPEN, keepCurTransition);
                mNoAnimActivities.remove(r);
            } else {
                mService.mWindowManager.prepareAppTransition(newTask
                        ? WindowManagerPolicy.TRANSIT_TASK_OPEN
                        : WindowManagerPolicy.TRANSIT_ACTIVITY_OPEN, keepCurTransition);
                mNoAnimActivities.remove(r);
            }
            mService.mWindowManager.addAppToken(
                    addPos, r, r.task.taskId, r.info.screenOrientation, r.fullscreen);
            boolean doShow = true;
            if (newTask) {
                // Even though this activity is starting fresh, we still need
                // to reset it to make sure we apply affinities to move any
                // existing activities from other tasks in to it.
                // If the caller has requested that the target task be
                // reset, then do so.
                if ((r.intent.getFlags()
                        &Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                    resetTaskIfNeededLocked(r, r);
                    doShow = topRunningNonDelayedActivityLocked(null) == r;
                }
            }
            if (SHOW_APP_STARTING_PREVIEW && doShow) {
                // Figure out if we are transitioning from another activity that is
                // "has the same starting icon" as the next one.  This allows the
                // window manager to keep the previous window it had previously
                // created, if it still had one.
                ActivityRecord prev = mResumedActivity;
                if (prev != null) {
                    // We don't want to reuse the previous starting preview if:
                    // (1) The current activity is in a different task.
                    if (prev.task != r.task) prev = null;
                    // (2) The current activity is already displayed.
                    else if (prev.nowVisible) prev = null;
                }
                mService.mWindowManager.setAppStartingWindow(
                        r, r.packageName, r.theme,
                        mService.compatibilityInfoForPackageLocked(
                                r.info.applicationInfo), r.nonLocalizedLabel,
                        r.labelRes, r.icon, r.windowFlags, prev, showStartingIcon);
            }
        } else {
            // If this is the first activity, don't do any fancy animations,
            // because there is nothing for it to animate on top of.
            mService.mWindowManager.addAppToken(addPos, r, r.task.taskId,
                    r.info.screenOrientation, r.fullscreen);
        }
        if (VALIDATE_TOKENS) {
            mService.mWindowManager.validateAppTokens(mHistory);
        }

        if (doResume) {
            resumeTopActivityLocked(null);
        }
    }
```




---

在Android系统全貌描述到了Zygote孵化了第一个进程是system_server进程，而且孵化第一个App进程是Launcher，也就是桌面App。


当点击桌面App的时候，发起进程就是Launcher所在的进程，启动远程进程，利用Binder发送消息给system_server进程；


在system_server进程中启动了N多服务，例如ActiivityManagerService，WindowManagerService等。启动进程的操作会先调用AMS.startProcessLocked方法，内部调用 Process.start(android.app.ActivityThread);而后通过socket通信告知Zygote进程fork子进程，即app进程。进程创建后将ActivityThread加载进去，执行ActivityThread.main()方法。


在app进程中，main方法会实例化ActivityThread，同时创建ApplicationThread，Looper，Hander对象，调用attach方法进行Binder通信，looper启动循环。attach方法内部获取ActivityManagerProxy对象，其实现了IActivityManager接口，作为客户端调用attachApplication(mAppThread)方法，将thread信息告知AMS。


在system_server进程中，AMS中会调用ActivityManagerNative.onTransact方法，真正的逻辑在服务端AMS.attachApplication方法中，内部调用AMS.attachApplicationLocked方法，方法的参数是IApplicationThread，在此处是ApplicationThreadProxy对象,用于跟前面通过Process.start()所创建的进程中ApplicationThread对象进行通信。
attachApplicationLocked方法会处理Provider, Activity, Service, Broadcast相应流程，调用ApplicationThreadProxy.bindApplication方法，通过Binder通信，传递给ApplicationThreadNative.onTransact方法。

在app进程中，真正的逻辑在ActivityThread.bindApplication方法中。bindApplication方法的主要功能是依次向主线程发送消息H.SET_CORE_SETTINGS
和H.BIND_APPLICATION。后续创建Application,Context等。Activity的回调也会是通过Binder通信，然后发送不同消息处理。



---

呼。。。好长的调用链。。。

![](https://ww1.sinaimg.cn/large/006tNc79gy1fd66mrnpu5g305k05kt90.gif)
