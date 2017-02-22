---
title: principle-analysis-on-android-activity-start-process
date: 2017-02-04 16:48:07
tags:
---


以startActivity为起始点开始进行分析，
startActivity调用了startActivityForResult：

```java

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
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }
        } else {
            mParent.startActivityFromChild(this, intent, requestCode);
        }
    }

```

可以看到，实际启动流程由Instrumentation的execStartActivity方法触发：

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

```
重点看`ActivityManagerNative.getDefault().startActivity()`这一段，
ActivityManagerNative.getDefault()得到ActivityManagerService的一个代理对象，

```java

    static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }

        return new ActivityManagerProxy(obj);
    }

    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };

```


接着调用ActivityManagerProxy的startActivity：


```java

public int startActivity(IApplicationThread caller, Intent intent,
            String resolvedType, Uri[] grantedUriPermissions, int grantedMode,
            IBinder resultTo, String resultWho,
            int requestCode, boolean onlyIfNeeded,
            boolean debug, String profileFile, ParcelFileDescriptor profileFd,
            boolean autoStopProfiler) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeTypedArray(grantedUriPermissions, 0);
        data.writeInt(grantedMode);
        data.writeStrongBinder(resultTo);
        data.writeString(resultWho);
        data.writeInt(requestCode);
        data.writeInt(onlyIfNeeded ? 1 : 0);
        data.writeInt(debug ? 1 : 0);
        data.writeString(profileFile);
        if (profileFd != null) {
            data.writeInt(1);
            profileFd.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        data.writeInt(autoStopProfiler ? 1 : 0);

        // 发起调用
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }

```


这里主要是通过ActivityManagerProxy内部的一个Binder代理对象mRemote向ActivityManagerService发起了一个**START_ACTIVITY_TRANSACTION**进程间通讯请求，

```java
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case START_ACTIVITY_TRANSACTION:
        {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();
            IApplicationThread app = ApplicationThreadNative.asInterface(b);
            Intent intent = Intent.CREATOR.createFromParcel(data);
            String resolvedType = data.readString();
            Uri[] grantedUriPermissions = data.createTypedArray(Uri.CREATOR);
            int grantedMode = data.readInt();
            IBinder resultTo = data.readStrongBinder();
            String resultWho = data.readString();    
            int requestCode = data.readInt();
            boolean onlyIfNeeded = data.readInt() != 0;
            boolean debug = data.readInt() != 0;
            String profileFile = data.readString();
            ParcelFileDescriptor profileFd = data.readInt() != 0
                    ? data.readFileDescriptor() : null;
            boolean autoStopProfiler = data.readInt() != 0;
            int result = startActivity(app, intent, resolvedType,
                    grantedUriPermissions, grantedMode, resultTo, resultWho,
                    requestCode, onlyIfNeeded, debug, profileFile, profileFd, autoStopProfiler);
            reply.writeNoException();
            reply.writeInt(result);
            return true;
        }
```

ActivityManagerService为ActivityManagerNative的实现类，它的startActivity用于处理START_ACTIVITY_TRANSACTION进程间通讯请求：


因此我们接下来切换到ActivityManagerService的startActivity中：

```java
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
```

接下来到了ActivityStack：

> 中间那个ActivityManagerNative.getDefault()返回的是ActivityManagerService的远程Binder对象。通过远程Binder对象调用ActivityManagerService的startActivity方法。 
接着startActivity 这个调用了很多层级的 函数。。这里省略 。。。到后面调用了一个 Ibind 的接口回到到app 进程中ApplicationThread类的scheduleLaunchActivity方法。然后调用 H 的sendMessage（） msg.what 为LAUNCH_ACTIVITY，紧接着 去执行performLaunchActivity（），performLaunchActivity里面的动作是 newActivity （真真正正的创建activity，尼玛绕了那么久，总于看到曙光了好辛苦）然后创建application 再去 执行activity 的attach 函数，attach 里面创建了window对象。（等下在onresume回调函数要显示到界面 需要用到），然后就是回调Activity的onCreate()。。。后面就是生命周期了 不多说了

```java
final int startActivityMayWait(IApplicationThread caller, int callingUid,
            Intent intent, String resolvedType, Uri[] grantedUriPermissions,
            int grantedMode, IBinder resultTo,
            String resultWho, int requestCode, boolean onlyIfNeeded,
            boolean debug, String profileFile, ParcelFileDescriptor profileFd,
            boolean autoStopProfiler, WaitResult outResult, Configuration config) {
        
        ...

        intent = new Intent(intent);

        // Collect information about the target of the Intent.
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
```



```java
    final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType,
            Uri[] grantedUriPermissions,
            int grantedMode, ActivityInfo aInfo, IBinder resultTo,
            String resultWho, int requestCode,
            int callingPid, int callingUid, boolean onlyIfNeeded,
            boolean componentSpecified, ActivityRecord[] outActivity) {


        ProcessRecord callerApp = null;
        if (caller != null) {
            callerApp = mService.getRecordForAppLocked(caller);
            if (callerApp != null) {
                callingPid = callerApp.pid;
                callingUid = callerApp.info.uid;
            } else {
                Slog.w(TAG, "Unable to find app for caller " + caller
                      + " (pid=" + callingPid + ") when starting: "
                      + intent.toString());
                err = START_PERMISSION_DENIED;
            }
        }

        ...

        ActivityRecord sourceRecord = null;
        ActivityRecord resultRecord = null;
        if (resultTo != null) {
            int index = indexOfTokenLocked(resultTo);
            if (DEBUG_RESULTS) Slog.v(
                TAG, "Will send result to " + resultTo + " (index " + index + ")");
            if (index >= 0) {
                sourceRecord = mHistory.get(index);
                if (requestCode >= 0 && !sourceRecord.finishing) {
                    resultRecord = sourceRecord;
                }
            }
        }

        
        ActivityRecord r = new ActivityRecord(mService, this, callerApp, callingUid,
                intent, resolvedType, aInfo, mService.mConfiguration,
                resultRecord, resultWho, requestCode, componentSpecified);
        if (outActivity != null) {
            outActivity[0] = r;
        }

        ...
        
        err = startActivityUncheckedLocked(r, sourceRecord,
                grantedUriPermissions, grantedMode, onlyIfNeeded, true);
        ...
        return err;
    }
```



```java
final int startActivityUncheckedLocked(ActivityRecord r,
            ActivityRecord sourceRecord, Uri[] grantedUriPermissions,
            int grantedMode, boolean onlyIfNeeded, boolean doResume) {
        final Intent intent = r.intent;
        final int callingUid = r.launchedFromUid;
        
        int launchFlags = intent.getFlags();
        
        // We'll invoke onUserLeaving before onPause only if the launching
        // activity did not explicitly state that this is an automated launch.
        mUserLeaving = (launchFlags&Intent.FLAG_ACTIVITY_NO_USER_ACTION) == 0;
        if (DEBUG_USER_LEAVING) Slog.v(TAG,
                "startActivity() => mUserLeaving=" + mUserLeaving);
        
        

        if (sourceRecord == null) {
            // This activity is not being started from another...  in this
            // case we -always- start a new task.
            if ((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
                Slog.w(TAG, "startActivity called from non-Activity context; forcing Intent.FLAG_ACTIVITY_NEW_TASK for: "
                      + intent);
                launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
            }
        } else if (sourceRecord.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
            // The original activity who is starting us is running as a single
            // instance...  this new activity it is starting must go on its
            // own task.
            launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
        } else if (r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE
                || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK) {
            // The activity being started is a single instance...  it always
            // gets launched into its own task.
            launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
        }

        if (r.resultTo != null && (launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
            // For whatever reason this activity is being launched into a new
            // task...  yet the caller has requested a result back.  Well, that
            // is pretty messed up, so instead immediately send back a cancel
            // and let the new task continue launched as normal without a
            // dependency on its originator.
            Slog.w(TAG, "Activity is launching as a new task, so cancelling activity result.");
            sendActivityResultLocked(-1,
                    r.resultTo, r.resultWho, r.requestCode,
                Activity.RESULT_CANCELED, null);
            r.resultTo = null;
        }

        boolean newTask = false;

        // Should this be considered a new task?
        if (r.resultTo == null && !addingToTask
                && (launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
            if (reuseTask == null) {
                // todo: should do better management of integers.
                mService.mCurTask++;
                if (mService.mCurTask <= 0) {
                    mService.mCurTask = 1;
                }
                r.setTask(new TaskRecord(mService.mCurTask, r.info, intent), null, true);
                if (DEBUG_TASKS) Slog.v(TAG, "Starting new activity " + r
                        + " in new task " + r.task);
            } else {
                r.setTask(reuseTask, reuseTask, true);
            }
            newTask = true;
            moveHomeToFrontFromLaunchLocked(launchFlags);
            
        } 

        ...

        startActivityLocked(r, newTask, doResume, keepCurTransition);
        return START_SUCCESS;
    }
```





```java

final ArrayList<ActivityRecord> mHistory = new ArrayList<ActivityRecord>();

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
                        
                        ...

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
        ...

        mHistory.add(addPos, r);
        r.putInHistory();
        r.frontOfTask = newTask;

        ...

        if (doResume) {
            resumeTopActivityLocked(null);
        }
    }
```

将目标Activity组件保存到ActivityStack内部的成员变量mHistory所描述的Activity堆栈中。


```java
final boolean resumeTopActivityLocked(ActivityRecord prev) {


        // Find the first activity that is not finishing.
        ActivityRecord next = topRunningActivityLocked(null);

        // Remember how we'll process this pause/resume situation, and ensure
        // that the state is reset however we wind up proceeding.
        final boolean userLeaving = mUserLeaving;
        mUserLeaving = false;

        ...
        
        // If the top activity is the resumed one, nothing to do.
        if (mResumedActivity == next && next.state == ActivityState.RESUMED) {
            ...
            return false;
        }

        // If we are sleeping, and there is no resumed activity, and the top
        // activity is paused, well that is the state we want.
        if ((mService.mSleeping || mService.mShuttingDown)
                && mLastPausedActivity == next && next.state == ActivityState.PAUSED) {
            ....
            return false;
        }
        
        ...

        // If we are currently pausing an activity, then don't do anything
        // until that is done.
        if (mPausingActivity != null) {
            if (DEBUG_SWITCH) Slog.v(TAG, "Skip resume: pausing=" + mPausingActivity);
            return false;
        }

        ...
        
        // We need to start pausing the current activity so the top one
        // can be resumed...
        if (mResumedActivity != null) {
            startPausingLocked(userLeaving, false);
            return true;
        }

        ...

        return true;
    }
```




```java
private final void startPausingLocked(boolean userLeaving, boolean uiSleeping) {
        
        ...

        ActivityRecord prev = mResumedActivity;
        
        ...

        mResumedActivity = null;
        mPausingActivity = prev;
        mLastPausedActivity = prev;
        prev.state = ActivityState.PAUSING;
        prev.task.touchActiveTime();
        prev.updateThumbnail(screenshotActivities(prev), null);

        mService.updateCpuStats();
        
        if (prev.app != null && prev.app.thread != null) {
          
            try {
               
                prev.app.thread.schedulePauseActivity(prev, prev.finishing, userLeaving, prev.configChangeFlags);
               ...
            } catch (Exception e) {
               ...
            }
        } else {
            mPausingActivity = null;
            mLastPausedActivity = null;
        }

        ...

        if (mPausingActivity != null) {
            ...

            // Schedule a pause timeout in case the app doesn't respond.
            // We don't give it much time because this directly impacts the
            // responsiveness seen by the user.
            Message msg = mHandler.obtainMessage(PAUSE_TIMEOUT_MSG);
            msg.obj = prev;
            mHandler.sendMessageDelayed(msg, PAUSE_TIMEOUT);
            ...

        } 
    }
```

ActivityRecord成员变量app为ProcessRecord类型，用于描述一个Activity组件所运行的在的应用程序进程，
其内部的thread为ApplicationThreadProxy类型



```java
    public final void schedulePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(token);
        data.writeInt(finished ? 1 : 0);
        data.writeInt(userLeaving ? 1 :0);
        data.writeInt(configChanges);
        mRemote.transact(SCHEDULE_PAUSE_ACTIVITY_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
```

```java
private class ApplicationThread extends ApplicationThreadNative {
        

        public final void schedulePauseActivity(IBinder token, boolean finished,
                boolean userLeaving, int configChanges) {
            queueOrSendMessage(
                    finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
                    token,
                    (userLeaving ? 1 : 0),
                    configChanges);
        }
```


```java
    private void queueOrSendMessage(int what, Object obj) {
        queueOrSendMessage(what, obj, 0, 0);
    }

    private void queueOrSendMessage(int what, Object obj, int arg1) {
        queueOrSendMessage(what, obj, arg1, 0);
    }

    private void queueOrSendMessage(int what, Object obj, int arg1, int arg2) {
        synchronized (this) {
            if (DEBUG_MESSAGES) Slog.v(
                TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
                + ": " + arg1 + " / " + obj);
            Message msg = Message.obtain();
            msg.what = what;
            msg.obj = obj;
            msg.arg1 = arg1;
            msg.arg2 = arg2;
            mH.sendMessage(msg);
        }
    }
```

mH为ActivityThread的成员变量，类型为H，继承了Handler类，主要用于处理应用程序进程的主线程的消息。

```java
.... 

public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + msg.what);
            switch (msg.what) {
                
                case PAUSE_ACTIVITY:
                    handlePauseActivity((IBinder)msg.obj, false, msg.arg1 != 0, msg.arg2);
                    maybeSnapshot();
                    break;

                    ...
            }
 }
```


```java
private void handlePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges) {
        ActivityClientRecord r = mActivities.get(token);
        if (r != null) {
            //Slog.v(TAG, "userLeaving=" + userLeaving + " handling pause of " + r);
            if (userLeaving) {
                performUserLeavingActivity(r);
            }

            r.activity.mConfigChangeFlags |= configChanges;
            performPauseActivity(token, finished, r.isPreHoneycomb());

            // Make sure any pending writes are now committed.
            if (r.isPreHoneycomb()) {
                QueuedWork.waitToFinish();
            }
            
            // Tell the activity manager we have paused.
            try {
                ActivityManagerNative.getDefault().activityPaused(token);
            } catch (RemoteException ex) {
            }
        }
    }
```


```java
    public final void activityPaused(IBinder token) {
        final long origId = Binder.clearCallingIdentity();
        mMainStack.activityPaused(token, false);
        Binder.restoreCallingIdentity(origId);
    }
```



```java
public void activityPaused(IBinder token) throws RemoteException
{
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(token);
    mRemote.transact(ACTIVITY_PAUSED_TRANSACTION, data, reply, 0);
    reply.readException();
    data.recycle();
    reply.recycle();
}
```

```java
    public final void activityPaused(IBinder token) {
        final long origId = Binder.clearCallingIdentity();
        mMainStack.activityPaused(token, false);
        Binder.restoreCallingIdentity(origId);
    }
```


```java
final void activityPaused(IBinder token, boolean timeout) {
        if (DEBUG_PAUSE) Slog.v(
            TAG, "Activity paused: token=" + token + ", timeout=" + timeout);

        ActivityRecord r = null;

        synchronized (mService) {
            int index = indexOfTokenLocked(token);
            if (index >= 0) {
                r = mHistory.get(index);
                mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
                if (mPausingActivity == r) {
                    if (DEBUG_STATES) Slog.v(TAG, "Moving to PAUSED: " + r
                            + (timeout ? " (due to timeout)" : " (pause complete)"));
                    r.state = ActivityState.PAUSED;
                    completePauseLocked();
                } else {
                    ...
                }
            }
        }
    }
```

上一个Activity成功pause



```java
    private final void completePauseLocked() {
        ActivityRecord prev = mPausingActivity;
        if (DEBUG_PAUSE) Slog.v(TAG, "Complete pause: " + prev);
        
        if (prev != null) {

            ...
            
            mPausingActivity = null;
        }

        if (!mService.isSleeping()) {
            resumeTopActivityLocked(prev);
        } else {
            ...
        }
        
       ...
    }
```




```java
    final boolean resumeTopActivityLocked(ActivityRecord prev) {
        // Find the first activity that is not finishing.
        ActivityRecord next = topRunningActivityLocked(null);

        // Remember how we'll process this pause/resume situation, and ensure
        // that the state is reset however we wind up proceeding.
        final boolean userLeaving = mUserLeaving;
        mUserLeaving = false;

        if (next == null) {
            // There are no more activities!  Let's just start up the
            // Launcher...
            if (mMainStack) {
                return mService.startHomeActivityLocked();
            }
        }

        next.delayedResume = false;
        
        // If the top activity is the resumed one, nothing to do.
        if (mResumedActivity == next && next.state == ActivityState.RESUMED) {
            // Make sure we have executed any pending transitions, since there
            // should be nothing left to do at this point.
            mService.mWindowManager.executeAppTransition();
            mNoAnimActivities.clear();
            return false;
        }

        // If we are sleeping, and there is no resumed activity, and the top
        // activity is paused, well that is the state we want.
        if ((mService.mSleeping || mService.mShuttingDown)
                && mLastPausedActivity == next && next.state == ActivityState.PAUSED) {
            // Make sure we have executed any pending transitions, since there
            // should be nothing left to do at this point.
            mService.mWindowManager.executeAppTransition();
            mNoAnimActivities.clear();
            return false;
        }
        
        // The activity may be waiting for stop, but that is no longer
        // appropriate for it.
        mStoppingActivities.remove(next);
        mGoingToSleepActivities.remove(next);
        next.sleeping = false;
        mWaitingVisibleActivities.remove(next);

        if (DEBUG_SWITCH) Slog.v(TAG, "Resuming " + next);

        // If we are currently pausing an activity, then don't do anything
        // until that is done.
        if (mPausingActivity != null) {
            if (DEBUG_SWITCH) Slog.v(TAG, "Skip resume: pausing=" + mPausingActivity);
            return false;
        }

        // Okay we are now going to start a switch, to 'next'.  We may first
        // have to pause the current activity, but this is an important point
        // where we have decided to go to 'next' so keep track of that.
        // XXX "App Redirected" dialog is getting too many false positives
        // at this point, so turn off for now.
        if (false) {
            if (mLastStartedActivity != null && !mLastStartedActivity.finishing) {
                long now = SystemClock.uptimeMillis();
                final boolean inTime = mLastStartedActivity.startTime != 0
                        && (mLastStartedActivity.startTime + START_WARN_TIME) >= now;
                final int lastUid = mLastStartedActivity.info.applicationInfo.uid;
                final int nextUid = next.info.applicationInfo.uid;
                if (inTime && lastUid != nextUid
                        && lastUid != next.launchedFromUid
                        && mService.checkPermission(
                                android.Manifest.permission.STOP_APP_SWITCHES,
                                -1, next.launchedFromUid)
                        != PackageManager.PERMISSION_GRANTED) {
                    mService.showLaunchWarningLocked(mLastStartedActivity, next);
                } else {
                    next.startTime = now;
                    mLastStartedActivity = next;
                }
            } else {
                next.startTime = SystemClock.uptimeMillis();
                mLastStartedActivity = next;
            }
        }
        
        // We need to start pausing the current activity so the top one
        // can be resumed...
        if (mResumedActivity != null) {
            if (DEBUG_SWITCH) Slog.v(TAG, "Skip resume: need to start pausing");
            startPausingLocked(userLeaving, false);
            return true;
        }

        if (prev != null && prev != next) {
            if (!prev.waitingVisible && next != null && !next.nowVisible) {
                prev.waitingVisible = true;
                mWaitingVisibleActivities.add(prev);
                if (DEBUG_SWITCH) Slog.v(
                        TAG, "Resuming top, waiting visible to hide: " + prev);
            } else {
                // The next activity is already visible, so hide the previous
                // activity's windows right now so we can show the new one ASAP.
                // We only do this if the previous is finishing, which should mean
                // it is on top of the one being resumed so hiding it quickly
                // is good.  Otherwise, we want to do the normal route of allowing
                // the resumed activity to be shown so we can decide if the
                // previous should actually be hidden depending on whether the
                // new one is found to be full-screen or not.
                if (prev.finishing) {
                    mService.mWindowManager.setAppVisibility(prev, false);
                    if (DEBUG_SWITCH) Slog.v(TAG, "Not waiting for visible to hide: "
                            + prev + ", waitingVisible="
                            + (prev != null ? prev.waitingVisible : null)
                            + ", nowVisible=" + next.nowVisible);
                } else {
                    if (DEBUG_SWITCH) Slog.v(TAG, "Previous already visible but still waiting to hide: "
                        + prev + ", waitingVisible="
                        + (prev != null ? prev.waitingVisible : null)
                        + ", nowVisible=" + next.nowVisible);
                }
            }
        }

        // Launching this app's activity, make sure the app is no longer
        // considered stopped.
        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    next.packageName, false);
        } catch (RemoteException e1) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, "Failed trying to unstop package "
                    + next.packageName + ": " + e);
        }

        // We are starting up the next activity, so tell the window manager
        // that the previous one will be hidden soon.  This way it can know
        // to ignore it when computing the desired screen orientation.
        if (prev != null) {
            if (prev.finishing) {
                if (DEBUG_TRANSITION) Slog.v(TAG,
                        "Prepare close transition: prev=" + prev);
                if (mNoAnimActivities.contains(prev)) {
                    mService.mWindowManager.prepareAppTransition(
                            WindowManagerPolicy.TRANSIT_NONE, false);
                } else {
                    mService.mWindowManager.prepareAppTransition(prev.task == next.task
                            ? WindowManagerPolicy.TRANSIT_ACTIVITY_CLOSE
                            : WindowManagerPolicy.TRANSIT_TASK_CLOSE, false);
                }
                mService.mWindowManager.setAppWillBeHidden(prev);
                mService.mWindowManager.setAppVisibility(prev, false);
            } else {
                if (DEBUG_TRANSITION) Slog.v(TAG,
                        "Prepare open transition: prev=" + prev);
                if (mNoAnimActivities.contains(next)) {
                    mService.mWindowManager.prepareAppTransition(
                            WindowManagerPolicy.TRANSIT_NONE, false);
                } else {
                    mService.mWindowManager.prepareAppTransition(prev.task == next.task
                            ? WindowManagerPolicy.TRANSIT_ACTIVITY_OPEN
                            : WindowManagerPolicy.TRANSIT_TASK_OPEN, false);
                }
            }
            if (false) {
                mService.mWindowManager.setAppWillBeHidden(prev);
                mService.mWindowManager.setAppVisibility(prev, false);
            }
        } else if (mHistory.size() > 1) {
            if (DEBUG_TRANSITION) Slog.v(TAG,
                    "Prepare open transition: no previous");
            if (mNoAnimActivities.contains(next)) {
                mService.mWindowManager.prepareAppTransition(
                        WindowManagerPolicy.TRANSIT_NONE, false);
            } else {
                mService.mWindowManager.prepareAppTransition(
                        WindowManagerPolicy.TRANSIT_ACTIVITY_OPEN, false);
            }
        }

        if (next.app != null && next.app.thread != null) {
            if (DEBUG_SWITCH) Slog.v(TAG, "Resume running: " + next);

            // This activity is now becoming visible.
            mService.mWindowManager.setAppVisibility(next, true);

            ActivityRecord lastResumedActivity = mResumedActivity;
            ActivityState lastState = next.state;

            mService.updateCpuStats();
            
            if (DEBUG_STATES) Slog.v(TAG, "Moving to RESUMED: " + next + " (in existing)");
            next.state = ActivityState.RESUMED;
            mResumedActivity = next;
            next.task.touchActiveTime();
            if (mMainStack) {
                mService.addRecentTaskLocked(next.task);
            }
            mService.updateLruProcessLocked(next.app, true, true);
            updateLRUListLocked(next);

            // Have the window manager re-evaluate the orientation of
            // the screen based on the new activity order.
            boolean updated = false;
            if (mMainStack) {
                synchronized (mService) {
                    Configuration config = mService.mWindowManager.updateOrientationFromAppTokens(
                            mService.mConfiguration,
                            next.mayFreezeScreenLocked(next.app) ? next : null);
                    if (config != null) {
                        next.frozenBeforeDestroy = true;
                    }
                    updated = mService.updateConfigurationLocked(config, next, false);
                }
            }
            if (!updated) {
                // The configuration update wasn't able to keep the existing
                // instance of the activity, and instead started a new one.
                // We should be all done, but let's just make sure our activity
                // is still at the top and schedule another run if something
                // weird happened.
                ActivityRecord nextNext = topRunningActivityLocked(null);
                if (DEBUG_SWITCH) Slog.i(TAG,
                        "Activity config changed during resume: " + next
                        + ", new next: " + nextNext);
                if (nextNext != next) {
                    // Do over!
                    mHandler.sendEmptyMessage(RESUME_TOP_ACTIVITY_MSG);
                }
                if (mMainStack) {
                    mService.setFocusedActivityLocked(next);
                }
                ensureActivitiesVisibleLocked(null, 0);
                mService.mWindowManager.executeAppTransition();
                mNoAnimActivities.clear();
                return true;
            }
            
            try {
                // Deliver all pending results.
                ArrayList a = next.results;
                if (a != null) {
                    final int N = a.size();
                    if (!next.finishing && N > 0) {
                        if (DEBUG_RESULTS) Slog.v(
                                TAG, "Delivering results to " + next
                                + ": " + a);
                        next.app.thread.scheduleSendResult(next, a);
                    }
                }

                if (next.newIntents != null) {
                    next.app.thread.scheduleNewIntent(next.newIntents, next);
                }

                EventLog.writeEvent(EventLogTags.AM_RESUME_ACTIVITY,
                        System.identityHashCode(next),
                        next.task.taskId, next.shortComponentName);
                
                next.sleeping = false;
                showAskCompatModeDialogLocked(next);
                next.app.pendingUiClean = true;
                next.app.thread.scheduleResumeActivity(next,
                        mService.isNextTransitionForward());
                
                checkReadyForSleepLocked();

            } catch (Exception e) {
                // Whoops, need to restart this activity!
                if (DEBUG_STATES) Slog.v(TAG, "Resume failed; resetting state to "
                        + lastState + ": " + next);
                next.state = lastState;
                mResumedActivity = lastResumedActivity;
                Slog.i(TAG, "Restarting because process died: " + next);
                if (!next.hasBeenLaunched) {
                    next.hasBeenLaunched = true;
                } else {
                    if (SHOW_APP_STARTING_PREVIEW && mMainStack) {
                        mService.mWindowManager.setAppStartingWindow(
                                next, next.packageName, next.theme,
                                mService.compatibilityInfoForPackageLocked(
                                        next.info.applicationInfo),
                                next.nonLocalizedLabel,
                                next.labelRes, next.icon, next.windowFlags,
                                null, true);
                    }
                }
                startSpecificActivityLocked(next, true, false);
                return true;
            }

            // From this point on, if something goes wrong there is no way
            // to recover the activity.
            try {
                next.visible = true;
                completeResumeLocked(next);
            } catch (Exception e) {
                // If any exception gets thrown, toss away this
                // activity and try the next one.
                Slog.w(TAG, "Exception thrown during resume of " + next, e);
                requestFinishActivityLocked(next, Activity.RESULT_CANCELED, null,
                        "resume-exception");
                return true;
            }

            // Didn't need to use the icicle, and it is now out of date.
            if (DEBUG_SAVED_STATE) Slog.i(TAG, "Resumed activity; didn't need icicle of: " + next);
            next.icicle = null;
            next.haveState = false;
            next.stopped = false;

        } else {
            // Whoops, need to restart this activity!
            if (!next.hasBeenLaunched) {
                next.hasBeenLaunched = true;
            } else {
                if (SHOW_APP_STARTING_PREVIEW) {
                    mService.mWindowManager.setAppStartingWindow(
                            next, next.packageName, next.theme,
                            mService.compatibilityInfoForPackageLocked(
                                    next.info.applicationInfo),
                            next.nonLocalizedLabel,
                            next.labelRes, next.icon, next.windowFlags,
                            null, true);
                }
                if (DEBUG_SWITCH) Slog.v(TAG, "Restarting: " + next);
            }
            startSpecificActivityLocked(next, true, true);
        }

        return true;
    }
```








