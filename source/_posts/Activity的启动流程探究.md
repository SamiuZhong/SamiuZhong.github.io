---
title: Activity的启动流程探究
keywords: Activity的启动流程探究
date: 2020-04-16 19:31:30
categories: Android
tags:
	- Android
	- 四大组件
	- Framework
---

## 概述

Activity的启动过程分为两种

- Activity跨进程启动
- Activity进程内启动

通常我们从桌面点击应用图标时就是跨进程启动Activity，而应用内普通的startActivity就是进程内启动。

这里我们大致说一下。

### Activity跨进程启动

在请求进程startActivity会请求AMS（Binder通信），AMS在这个过程中主要做了一下几点事情

1. 解析Activity信息（Manifest里面的那些参数等）
2. 处理启动参数
3. 启动目标进程（Zygote）
4. 绑定新进程

AMS在完成了以上的这些操作之后就会把请求转交到ApplicationThread（Binder通信）（ApplicationThread是ActivityThread的内部类），后续由ApplicationThread做相应的处理，直到我们熟知的Activity生命周期。

### Activity进程内启动

进程内启动activity也会请求AMS，过程和上面相仿，只是AMS的处理过程会少了3、4两个步骤，也就是说不会启动新进程了，然后转到当前进程的ApplicationThread继续做处理。

## Activity跨进程启动的源码探究

### Launcher请求AMS

当我们在桌面点击应用程序的图标时，会调用Launcher的startActivitySafely方法

```java
public boolean startActivitySafely(View v, Intent intent, ItemInfo item) {
    ...
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);	//comment 1
    if (v != null) {
        intent.setSourceBounds(getViewBounds(v));
    }
    try {
        if (Utilities.ATLEAST_MARSHMALLOW
                && (item instanceof ShortcutInfo)
                && (item.itemType == Favorites.ITEM_TYPE_SHORTCUT
                || item.itemType == Favorites.ITEM_TYPE_DEEP_SHORTCUT)
                && !((ShortcutInfo) item).isPromise()) {
            startShortcutIntentSafely(intent, optsBundle, item);
        } else if (user == null || user.equals(Process.myUserHandle())) {
            startActivity(intent, optsBundle);	//comment 2
        } else {
            LauncherAppsCompat.getInstance(this).startActivityForProfile(
                    intent.getComponent(), user, intent.getSourceBounds(), optsBundle);
        }
        return true;
    } catch (ActivityNotFoundException|SecurityException e) {
        Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
        Log.e(TAG, "Unable to launch. tag=" + item + " intent=" + intent, e);
    }
    return false;
}
```

可以看到

- 在comment 1处我们将Flag设置为了Intent.FLAG_ACTIVITY_NEW_TASK，这样根Activity就会在新的任务栈中启动
- 在comment 2处我们调用的startActivity方法，这个方法会在Activity中实现，我们去看下

```java
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        startActivityForResult(intent, -1);
    }
}
```

显然，这里又将烂摊子交给了startActivityForResult方法，第二个参数-1表示Launcher不想晓得Activity启动的结果。那我们又接到去看看这个方法咯

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
                                   @Nullable Bundle options) {
    if (mParent == null) {	//comment 1
        options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                        this, mMainThread.getApplicationThread(), mToken, this,
                        intent, requestCode, options);
        ...
    } else {
        ...
    }
}
```

commaent 1处的mParent指当前Activity的父类。因为目前根Activity还没创建出来嘛，所以这里判断条件成立，会进if语句里面去。接着就会调用Instrumentation的execStartActivity方法。这个Instrumentation主要用来监控应用程序和系统的交互。那我们又去看看它的这个execStartActivity方法嘛

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    ...
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
        int result = ActivityManager.getService()	//comment 1
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

这里可以看到我们在comment 1处首先调用了ActivityManager的getService方法来获取AMS的代理对象，接着调用了它的startActivity方法。

所以从上面这段代码我们得知Instrumentation的execStartActivity最终调用的是AMS的startActivity方法。

### AMS到ApplicationThread的调用过程

上面我们通过Launcher请求AMS后，代码逻辑已经走到了AMS当中去了。

那我们又去看看AMS当中又是个什么鬼逻辑嘛。

先打开AMS的startActivity方法

```java
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
                               Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
                               int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}
```

这个龟儿子又把问题抛给了startActivityAsUser，那我们又去看看这个方法嘛

```java
@Override
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
                                     Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
                                     int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
    enforceNotIsolatedCaller("startActivity");	//comment 1
    userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
            userId, false, ALLOW_FULL_ONLY, "startActivity", null);	//comment 2
    return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
            resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
            profilerInfo, null, null, bOptions, false, userId, null, null,
            "startActivityAsUser");
}
```

- 这里在comment 1处判断调用者的线程是否被隔离，如果被隔离了则抛出SecurityException异常
- 在comment 2处检查调用者是否有权限，如果没有权限同样也会抛出上面哪个异常
- 最后调用了ActivityStarter的startActivityLocked方法。那我们又去看看这个方法嘛

```java
int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
                        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
                        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
                        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
                        ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
                        ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container,
                        TaskRecord inTask, String reason) {

    if (TextUtils.isEmpty(reason)) {	//comment 1
        throw new IllegalArgumentException("Need to specify a reason.");
    }
    mLastStartReason = reason;
    mLastStartActivityTimeMs = System.currentTimeMillis();
    mLastStartActivityRecord[0] = null;

    mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
            aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
            callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
            options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
            container, inTask);	//comment 2

    if (outActivity != null) {
        outActivity[0] = mLastStartActivityRecord[0];
    }
    return mLastStartActivityResult;
}
```

- 在comment 1处判断启动的理由不为空，否则就会抛出IllegalArgumentException异常。
- 紧接着又在comment 2调用了startActivity方法，那我们又去看看这个狗日的方法嘛。

```java
private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
                          String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
                          IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                          IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
                          String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
                          ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
                          ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container,
                          TaskRecord inTask) {
    int err = ActivityManager.START_SUCCESS;
    final Bundle verificationBundle
            = options != null ? options.popAppVerificationBundle() : null;
    ProcessRecord callerApp = null;
    if (caller != null) {	//comment 1
        callerApp = mService.getRecordForAppLocked(caller);	//comment 2
        if (callerApp != null) {
            callingPid = callerApp.pid;
            callingUid = callerApp.info.uid;
        } else {
            Slog.w(TAG, "Unable to find app for caller " + caller
                    + " (pid=" + callingPid + ") when starting: "
                    + intent.toString());
            err = ActivityManager.START_PERMISSION_DENIED;
        }
    }
    ...
    ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
            callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
            resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
            mSupervisor, container, options, sourceRecord);	//comment 3
    if (outActivity != null) {
        outActivity[0] = r;	//comment 3
    }
    ...
    doPendingActivityLaunchesLocked(false);
    return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags, true,
            options, inTask, outActivity);
}
```

- 首先在comment 1处判断一个caller是否为null，这个call是一路传过来的，指的是Launcher所在的应用程序进程的ApplicationThread对象
- 接着在comment 2调用AMS的这个方法代表Launcher进程的callerApp对象，它是ProcessRecord类型的，用于描述一个应用程序进程。同理我们可知ActivityRecord用与描述一个Activity（即记录一个Activity的所有信息）。
- 然后在comment 3创建了一个ActivityRecord用来描述即将要启动的这个Activity
- 随后又在comment 4将这个Record赋值给outActivity这个东西，这个outActivity会作为参数一直传下去
- 最后又调用了一个狗日的startActivity方法，跟过去看看

```java
private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                          IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                          int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                          ActivityRecord[] outActivity) {
    int result = START_CANCELED;
    try {
        mService.mWindowManager.deferSurfaceLayout();
        result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, doResume, options, inTask, outActivity);	//commant 1
    }
	...
    return result;
}
```

可以发现它在commant 1处又调用了一个startActivityUnchecked方法，又跟过去看

```java
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
                                   IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                                   int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                                   ActivityRecord[] outActivity) {
    ...
    if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
            && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
        newTask = true;
        result = setTaskFromReuseOrCreateNewTask(
                taskToAffiliate, preferredLaunchStackId, topStack);	//commant 1
    } else if (mSourceRecord != null) {
        result = setTaskFromSourceRecord();
    } else if (mInTask != null) {
        result = setTaskFromInTask();
    } else {
        ...
        setTaskToCurrentTopOrCreateNewTask();
    }
    ...
    if (mDoResume) {
        final ActivityRecord topTaskActivity =
                mStartActivity.getTask().topRunningActivityLocked();
        if (!mTargetStack.isFocusable()
                || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                && mStartActivity != topTaskActivity)) {
            ...
        } else {
            if (mTargetStack.isFocusable() && !mSupervisor.isFocusedStack(mTargetStack)) {
                mTargetStack.moveToFront("startActivityUnchecked");
            }
            mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                    mOptions);	//commant 2
        }
    } else {
        mTargetStack.addRecentActivityLocked(mStartActivity);
    }
    ...
}
```

可以看出这个startActivityUnchecked方法主要是用来处理与栈管理相关的逻辑。

- 在commant 1处我们创建了一个新的TaskRecord用来描述一个Activity任务栈
- 然后在commant 2处又调用了ActivityStackSupervisor的resumeFocusedStackTopActivityLocked方法，我们去看看

```java
boolean resumeFocusedStackTopActivityLocked(
        ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
    if (targetStack != null && isFocusedStack(targetStack)) {
        return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
    }
    final ActivityRecord r = mFocusedStack.topRunningActivityLocked();	//commant 1
    if (r == null || r.state != RESUMED) {	//commant 2
        mFocusedStack.resumeTopActivityUncheckedLocked(null, null);	//commant 3
    } else if (r.state == RESUMED) {
        mFocusedStack.executeAppTransition(targetOptions);
    }
    return false;
}
```

- 在commant 1处调用了一个方法获得__要启动的Activity所在的栈的栈顶不是处于停止状态的ActivityRecord__
- 在commant 2处进行一个判断，紧接着就在commant 3调用了ActivityStack的一个方法，跟上看看

```java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    if (mStackSupervisor.inResumeTopActivity) {
        return false;
    }
    boolean result = false;
    try {
        mStackSupervisor.inResumeTopActivity = true;
        result = resumeTopActivityInnerLocked(prev, options);	//commant 1
    } finally {
        mStackSupervisor.inResumeTopActivity = false;
    }
    mStackSupervisor.checkReadyForSleepLocked();
    return result;
}
```

这里又调用了一个方法

```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    ...
    if (next.app != null && next.app.thread != null) {
        ...
        try {
            ...
        } catch (Exception e) {
            ...
            mStackSupervisor.startSpecificActivityLocked(next, true, false);	//commant 1
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return true;
        }
        ...
    } else {
        ...
    }
    if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
    return true;
}
```

从commant 1我们可以看到这里又调回去了ActivityStackSupervisor的一个狗日的方法，又看嘛

```java
void startSpecificActivityLocked(ActivityRecord r,
                                 boolean andResume, boolean checkConfig) {
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,
            r.info.applicationInfo.uid, true);	//commant 1
    r.getStack().setLaunchTime(r);
    if (app != null && app.thread != null) {	//commant 2
        try {
            if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                    || !"android".equals(r.info.packageName)) {
                app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                        mService.mProcessStats);
            }
            realStartActivityLocked(r, app, andResume, checkConfig);	//commant 3
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }
    }
    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
            "activity", r.intent.getComponent(), false, false, true);
}   
```

- 在commant 1处获取即将启动的Activity所在的应用程序进程
- 在commant 2处进行一个判断，如果这个进程不为null的话就在commant 3调用一个方法，这个方法的第二个参数app代表将要启动的Activity所在的应用程序进程的ProcessRecord。那又去看这个方法嘛

```java
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
                                      boolean andResume, boolean checkConfig) throws RemoteException {
    ...
    try {
        ...
        app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                System.identityHashCode(r), r.info,
                mergedConfiguration.getGlobalConfiguration(),
                mergedConfiguration.getOverrideConfiguration(), r.compat,
                r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                r.persistentState, results, newIntents, !andResume,
                mService.isNextTransitionForward(), profilerInfo);
        ...
    } catch (RemoteException e) {
        ...
    }
    ...
    return true;
}
```

这段代码的主要逻辑指的就是要在__目标应用程序进程中__启动Activity

### ActivityThread启动Activity的过程

接到上面的流程，我们来查看ApplicationThread的scheduleLaunchActivity方法，这里的ApplicationThread是ActivityThread的内部类，ActivityThread管理当前应用程序进程的主线程

```java
@Override
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                                         ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                                         CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                                         int procState, Bundle state, PersistableBundle persistentState,
                                         List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                                         boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
    updateProcessState(procState, false);
    ActivityClientRecord r = new ActivityClientRecord();
    r.token = token;
    r.ident = ident;
    r.intent = intent;
    r.referrer = referrer;
    ...
    updatePendingConfiguration(curConfig);
    sendMessage(H.LAUNCH_ACTIVITY, r);
}
```

- 可以看到这个方法内部将启动Activity的参数封装成ActivityClientRecord，然后通过sendMessage方法向H类发送LAUNCH_ACTIVITY的消息，并将ActivityClientRecord传递了过去
- sendMessage有多个重载方法，一顿操作后最终会调用到下面这个重载方法

```java
private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
    if (DEBUG_MESSAGES) Slog.v(
            TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
                    + ": " + arg1 + " / " + obj);
    Message msg = Message.obtain();
    msg.what = what;
    msg.obj = obj;
    msg.arg1 = arg1;
    msg.arg2 = arg2;
    if (async) {
        msg.setAsynchronous(true);
    }
    mH.sendMessage(msg);
}
```

- 这里的H是一个ActivityThread的内部类并且继承自Handler，用来管理应用程序进程中主线程的消息
- 因为ApplicationThread是一个Binder，它的调用逻辑运行在Binder线程池中，所以这里需要通过sendMessage将代码的逻辑切换到主线程中。我们来看看H里面的逻辑

```java
private class H extends Handler {
    ...
    public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
            case LAUNCH_ACTIVITY: {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                final ActivityClientRecord r = (ActivityClientRecord) msg.obj;	//comment 1

                r.packageInfo = getPackageInfoNoCheck(
                        r.activityInfo.applicationInfo, r.compatInfo);	//comment 2
                handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");  //comment 3
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            } break;
            case RELAUNCH_ACTIVITY: {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityRestart");
                ActivityClientRecord r = (ActivityClientRecord)msg.obj;
                handleRelaunchActivity(r);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            } break;
            ...
        }
        ...
    }
    ...
}
```

- 这里先在comment 1处将传过来的msg的成员变量obj转换为ActivityClientRecord
- 然后在comment 2处通过getPackageInfoNoCheck方法获得LoadedApk类型的对象并赋值给ActivityClientRecord的成员变量packageInfo。应用程序进程要启动Activity时需要将该Activity所属的APK加载进来，而LoadedApk就是用来描述已加载的APK文件的。
- 在comment 3又调用了一个狗日的handleLaunchActivity方法，又去看看

 ```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    ...
    WindowManagerGlobal.initialize();
    //启动Activity
    Activity a = performLaunchActivity(r, customIntent);	//comment 1
    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        reportSizeConfigurations(r);
        Bundle oldState = r.state;
        //将Activity的状态设置为Resume
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);	//comment 2
        if (!r.activity.mFinished && r.startsNotResumed) {
            performPauseActivityIfNeeded(r, reason);
            if (r.isPreHoneycomb()) {
                r.state = oldState;
            }
        }
    } else {
        try {
            //停止Activity启动
            ActivityManager.getService()
                    .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                            Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
}
 ```

- 可以看到在comment 1处调用了performLaunchActivity方法来启动Activity
- comment 2处的代码用来将Activity的状态设置为Resume
- 如果Activity为null则会通知AMS停止启动Activity

那我们又去看看performLaunchActivity这个方法干了啥

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    //获取ActivityInfo类
    ActivityInfo aInfo = r.activityInfo;    //comment 1
    if (r.packageInfo == null) {
        //获取APK文件的描述类LoadApk
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                Context.CONTEXT_INCLUDE_CODE);  //comment 2
    }
    ComponentName component = r.intent.getComponent();  //comment 3
    ...
    //创建要启动Activity的上下文环境
    ContextImpl appContext = createBaseContextForActivity(r);   //comment 4
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();
        //用类加载器来创建该Activity的实例
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);    //comment 5
        ...
    } catch (Exception e) {
        ...
    }
    try {
        //创建Application
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);   //comment 6
        ...
        if (activity != null) {
            ...
            //初始化Activity
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback);   //comment 7
            ...
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);    //comment 8
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            ...
        }
        r.paused = true;
        mActivities.put(r.token, r);
    } catch (SuperNotCalledException e) {
        throw e;
    } catch (Exception e) {
        ...
    }
    return activity;
}
```

- 这里在comment 1获取ActivityInfo，用于存储代码以及AndroidManifest设置的Activity和Reciever节点信息，比如Activity的theme和launchMode
- 在comment 2获取APK文件的描述类LoadedApk
- 在comment 3处获取要启动的Activity的ComponentName类，该类中保存了该Activity的包名和类名
- comment 4处用来创建要启动Activity的上下文环境
- comment 5处根据ComponentName中存储的Activity类名，用类加载器来创建该Activity的实例
- comment 6处用来创建Application，makeApplication方法内部会调用Application的onCreate方法
- comment 7处调用Activity的attach方法初始化Activity，在attach方法中会创建Window对象(PhoneWindow)并与Activity自身进行关联
- comment 8处调用了Instrumentation的callActivityOnCreate方法来启动Activity，去看看

```java
public void callActivityOnCreate(Activity activity, Bundle icicle,
                                 PersistableBundle persistentState) {
    prePerformCreate(activity);
    activity.performCreate(icicle, persistentState);	//comment 1
    postPerformCreate(activity);
}
```

可以看到这里在comment 1处调用了Activity的performCreate方法，点开看看

```java
final void performCreate(Bundle icicle) {
    performCreate(icicle, null);
}

final void performCreate(Bundle icicle, PersistableBundle persistentState) {
    mCanEnterPictureInPicture = true;
    restoreHasCurrentPermissionRequest(icicle);
    if (persistentState != null) {
        onCreate(icicle, persistentState);	//comment 1
    } else {
        onCreate(icicle);
    }
    mActivityTransitionState.readState(icicle);

    mVisibleFromClient = !mWindow.getWindowStyle().getBoolean(
            com.android.internal.R.styleable.Window_windowNoDisplay, false);
    mFragments.dispatchActivityCreated();
    mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
}
```

可以看到，这里在comment 1处调用了onCreate方法，也就也就走到我们熟悉的生命周期了。

## 参考资料

__1. 《Android进阶解密》__
