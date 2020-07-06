### 1. App进程发起请求

Activity的启动，从Activity的startActivity开始分析：

```java
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }

    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }

    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode) {
        startActivityForResult(intent, requestCode, null);
    }

```

可以看出，最终都是执行`startActivityForResult(Intent intent, int requestCode,Bundle options)`

```java
     public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            //1.调用Instrumentation.execStartActivity
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                mStartedActivity = true;
            }
            cancelInputsAndStartExitTransition(options);
        } else {
            ...
        }
    }
```

这里的mParent指Activity的父Activity，在没有Fragment之前，一般用Activity包裹Activity的方式，实现现在的一些四个分页的界面。目前基本是使用Fragment，当然不管是有没有mParent，最终也是调用`Instrumentation#execStartActivity`启动Activity

#### 1.1 Instrumentation#execStartActivity

```java
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        ...
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            //获取ActivityTaskManagerService调用它的startActivity方法
            int result = ActivityTaskManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            //检查Activity是否启动成功
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

`Instrumentation#execStartActivity`中通过请求系统服务来启动Activity，在Android10中启动Activity的责任由ActivityManagerService交给了**ActivityTaskManagerService**，这里首先通过aidl的方式得到ActivityTaskManagerService的Binder代理对象。

```java
    public static IActivityTaskManager getService() {
        return IActivityTaskManagerSingleton.get();
    }

    private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
            new Singleton<IActivityTaskManager>() {
                @Override
                protected IActivityTaskManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                    return IActivityTaskManager.Stub.asInterface(b);
                }
            };
```

### 2. ATMS处理请求

#### 2.0 介绍一下几个基本的类

![](https://img-blog.csdnimg.cn/20191206221732510.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTExMDEwMzExNw==,size_16,color_FFFFFF,t_70)

- ActivityRecord 是 Activity 管理的最小单位，它对应着一个用户界面，是 AMS调度 Activity的基本单位。
- TaskRecord 是一个栈式管理结构，每一个 TaskRecord 都可能存在一个或多个 ActivityRecord，栈顶的ActivityRecord 表示当前可见的界面。启动 Activity时，需要找到 Activity的宿主任务，如果不存在，则需要新建一个，也就是说所有的 ActivityRecord都必须有宿主。
- ActivityStack 是一个栈式管理结构，每一个 ActivityStack 都可能存在一个或多个 TaskRecord，栈顶的TaskRecord 表示当前可见的任务。
- ActivityStackSupervisor 管理着多个 ActivityStack，但当前只会有一个获取焦点 (Focused)的 ActivityStack。
- ProcessRecord 记录着属于一个进程的所有 ActivityRecord，运行在不同 TaskRecord 中的 ActivityRecord 可能是属于同一个 ProcessRecord。AMS采用 ProcessRecord 这个数据结构来维护进程运行时的状态信息，当创建系统进程 (system_process) 或应用进程的时候，就会通过 AMS初始化一个 ProcessRecord。

#### 2.1 ActivityTaskManagerService#startActivity

在上一步中获取了ActivityTaskManagerService的Binder代理对象，通过Binder调用远程服务ActivityTaskManagerService的startActivity方法：

```java
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,resultWho, requestCode, startFlags, profilerInfo, bOptions,UserHandle.getCallingUserId());
    }

```

startActivity方法接受10个参数：

- caller：当前应用的ApplicationThread对象
- callingPackage: 调用当前ContextImpl.getBasePackageName(),获取当前Activity所在包名
- intent: 这便是启动Activity时,传递过来的参数
- resolvedType: null
- resultTo: 当前的Activity的mToken参数，是一个IBinder对象
- resultWho: null
- requestCode = -1
- startFlags = 0
- profilerInfo = null
- options = null

#### 2.2 ActivityTaskManagerService#startActivityAsUser

```java
    int startActivityAsUser(IApplicationThread caller, String callingPackage,
Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
boolean validateIncomingUser) {
        enforceNotIsolatedCaller("startActivityAsUser");

        userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        // 获取ActivityStartContoller，通过建造者模式赋值
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setMayWait(userId)
                .execute();

    }
```

startActivityAsUser中，通过getActivityStartController得到的是ActivityStartContoller。通过ActivityStartController#obtainStarter得到的是ActivityStarter。最终会调用ActivityStarter#execute

##### 2.2.1 ActivityStartController

ActivityTaskManagerService通过getActivityStartController()得到ActivityStartController对象来开启Activity，ActivityStartController是ActivityTaskManagerService的成员函数，它是在ActivityTaskManagerService初始化方法initialize函数中被创建，而initialize函数是在ActivityManagerService的构造方法中被调用的。

```
    public void initialize(IntentFirewall intentFirewall, PendingIntentController intentController,
            Looper looper) {
        ...
        mActivityStartController = new ActivityStartController(this);
        ...
    }

```

ActivityStartController是启动Activity的控制器，它的主要目标是接受启动Activity的请求，并将一些列的Activity准备好交给ActivityStarter处理，它还负责处理围绕Activity启动的逻辑，但不一定会影响Activity启动。例如包括电源提示管理、通过Activity等待列表处理 以及记录home Activity启动。

##### 2.2.2 ActivityStarter

**ActivityStartController只是做好围绕启动Activity的工作，而真正的启动Activity的任务交给了ActivityStarter。**ActivityStartController通过obtainStarter得到ActivityStarter：

```java
    ActivityStarter obtainStarter(Intent intent, String reason) {
        return mFactory.obtain().setIntent(intent).setReason(reason);
    }
```

这里通过ActivityStarter的工厂类DefaultFactory得到ActivityStarter对象：

```java
        //ActivityStarter.DefaultFactory
		@Override
        public ActivityStarter obtain() {
            ActivityStarter starter = mStarterPool.acquire();
            if (starter == null) {
                starter = new ActivityStarter(mController, mService, mSupervisor, mInterceptor);
            }
            return starter;
        }

```

ActivityStarter创建需要四个参数：

```java
        private ActivityStartController mController;
        private ActivityTaskManagerService mService;
        private ActivityStackSupervisor mSupervisor;
        private ActivityStartInterceptor mInterceptor;
```

ActivityStartController与ActivityTaskManagerService已经知道，ActivityStackSupervisor和ActivityStartInterceptor接下来随着Activity的启动，我们继续分析。

2.3 ActivityStarter#execute

得到ActivityStarter对象并且设置一系列参数后，最终调用了ActivityStarter的execute()方法：

```java
    int execute() {
        try {
            if (mRequest.mayWait) {
                return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                        mRequest.callingPackage, mRequest.realCallingPid, mRequest.realCallingUid,
                        mRequest.intent, mRequest.resolvedType,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                        mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                        mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                        mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup,
                        mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
            } else {
                return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                        mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                        mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                        mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                        mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                        mRequest.outActivity, mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup,
                        mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
            }
        } finally {
            onExecutionComplete();
        }
    }

```

由于之前在startActivityAsUser函数中，调用了ActivityStarter的setMayWait()会设置mRequest.mayWait为true，所以这里会进入startActivityMayWait函数中

#### 2.4 ActivityStarter#startActivityMayWait

```java
    private int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, int requestRealCallingPid, int requestRealCallingUid,
            Intent intent, String resolvedType, IVoiceInteractionSession voiceSession,
            IVoiceInteractor voiceInteractor, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, WaitResult outResult,
            Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
            int userId, TaskRecord inTask, String reason,
            boolean allowPendingRemoteAnimationRegistryLookup,
            PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
        ...
            
        final Intent ephemeralIntent = new Intent(intent);
        // 创建新的Intent对象，即便intent被修改也不受影响
        intent = new Intent(intent);
		...
		//通过ActivityStackSupervisor解析Intent的信息
        //ResoleveInfo中包含要启动的Activity信息，即ActivityInfo对象
        ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId,
                0 /* matchFlags */,
                        computeResolveFilterUid(
                                callingUid, realCallingUid, mRequest.filterCallingUid));
        ...
        //收集Intent所指向的Activity信息
        //如果rInfo不为null则aInfo则直接取rInfo.activityInfo
        ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);

        synchronized (mService.mGlobalLock) {
            //得到当前显示的Activity所属的ActivityStack
            final ActivityStack stack = mRootActivityContainer.getTopDisplayFocusedStack();
            stack.mConfigWillChange = globalConfig != null
                    && mService.getGlobalConfiguration().diff(globalConfig) != 0;
            if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION,
                    "Starting activity when config will change = " + stack.mConfigWillChange);

            final long origId = Binder.clearCallingIdentity();

            ...

            final ActivityRecord[] outRecord = new ActivityRecord[1];
            //调用startActivity
            int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
                    callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
                    ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
                    allowPendingRemoteAnimationRegistryLookup, originatingPendingIntent,
                    allowBackgroundActivityStart);

            Binder.restoreCallingIdentity(origId);

            ...
            return res;
        }
    }


```

##### 2.4.1 mSupervisor.resolveIntent

startActivityMayWait中会调用mSupervisor.resolveIntent，其目的是找到相应的组件，mSupervisor的类型是ActivityStackSupervisor，在Android10之前，主要是由这个类启动Activity，Android10之后，将逐渐将它的功能分离给相应的类，例如将与层次结构相关的内容移动到RootWindowContainer。当然，目前这个类还没有一下子被取代。看它的resolveIntent：

```
    ResolveInfo resolveIntent(Intent intent, String resolvedType, int userId, int flags,
            int filterCallingUid) {
        try {
            ...
            final long token = Binder.clearCallingIdentity();
            try {
                return mService.getPackageManagerInternalLocked().resolveIntent(
                        intent, resolvedType, modifiedFlags, userId, true, filterCallingUid);
            } finally {
                Binder.restoreCallingIdentity(token);
            }
        } finally {
            Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
        }
    }

```

resolveIntent会获取PackageManagerService的本地服务类PackageManagerInternalImpl，它是在PackageManagerService构造时，创建并加入到LocalServices中的。

```java
    @Override
    public ResolveInfo resolveIntent(Intent intent, String resolvedType,
            int flags, int userId) {
        return resolveIntentInternal(intent, resolvedType, flags, userId, false,
                Binder.getCallingUid());
    }

    private ResolveInfo resolveIntentInternal(Intent intent, String resolvedType,
            int flags, int userId, boolean resolveForStart, int filterCallingUid) {
        try {
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "resolveIntent");

            if (!sUserManager.exists(userId)) return null;
            final int callingUid = Binder.getCallingUid();
            flags = updateFlagsForResolve(flags, userId, intent, filterCallingUid, resolveForStart);
            mPermissionManager.enforceCrossUserPermission(callingUid, userId,
                    false /*requireFullPermission*/, false /*checkShell*/, "resolve intent");

            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "queryIntentActivities");
            //查询符合的组件
            final List<ResolveInfo> query = queryIntentActivitiesInternal(intent, resolvedType,
                    flags, filterCallingUid, userId, resolveForStart, true /*allowDynamicSplits*/);
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
			//选择最佳的组件
            final ResolveInfo bestChoice =
                    chooseBestActivity(intent, resolvedType, flags, query, userId);
            return bestChoice;
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
    }

```



#### 2.5 ActivityStarter#startActivity

startActivityMayWait最终会调用一系列的startActivity方法

```java
    private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            SafeActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,ActivityRecord[] outActivity, TaskRecord inTask, String reason,boolean allowPendingRemoteAnimationRegistryLookup,PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
        ...
        mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
inTask, allowPendingRemoteAnimationRegistryLookup, originatingPendingIntent,allowBackgroundActivityStart);

        if (outActivity != null) {
            outActivity[0] = mLastStartActivityRecord[0];
        }
        return getExternalResult(mLastStartActivityResult);
    }

```



```java
    private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            SafeActivityOptions options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            TaskRecord inTask, boolean allowPendingRemoteAnimationRegistryLookup,
            PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
        mSupervisor.getActivityMetricsLogger().notifyActivityLaunching(intent);
        int err = ActivityManager.START_SUCCESS;
        // Pull the optional Ephemeral Installer-only bundle out of the options early.
        final Bundle verificationBundle
                = options != null ? options.popAppVerificationBundle() : null;
		//得到进程控制器
        WindowProcessController callerApp = null;
        if (caller != null) {
            callerApp = mService.getProcessController(caller);
            if (callerApp != null) {
                callingPid = callerApp.getPid();
                callingUid = callerApp.mInfo.uid;
            } else {
                Slog.w(TAG, "Unable to find app for caller " + caller
                        + " (pid=" + callingPid + ") when starting: "
                        + intent.toString());
                err = ActivityManager.START_PERMISSION_DENIED;
            }
        }

        final int userId = aInfo != null && aInfo.applicationInfo != null
                ? UserHandle.getUserId(aInfo.applicationInfo.uid) : 0;

        if (err == ActivityManager.START_SUCCESS) {
            Slog.i(TAG, "START u" + userId + " {" + intent.toShortString(true, true, true, false)
                    + "} from uid " + callingUid);
        }

        ActivityRecord sourceRecord = null;
        ActivityRecord resultRecord = null;
        if (resultTo != null) {
            //获取当前Activity的ActivityRecord，即发起启动请求的Activity
            sourceRecord = mRootActivityContainer.isInAnyStack(resultTo);
            ...
        }

		...

   		//创建要启动的Activity的ActivityRecord对象
        ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,mSupervisor, checkedOptions, sourceRecord);
        if (outActivity != null) {
            outActivity[0] = r;
        }

        if (r.appTimeTracker == null && sourceRecord != null) {
            r.appTimeTracker = sourceRecord.appTimeTracker;
        }
		//得到当前正在显示的ActivityStack
        final ActivityStack stack = mRootActivityContainer.getTopDisplayFocusedStack();
        if (voiceSession == null && (stack.getResumedActivity() == null
                || stack.getResumedActivity().info.applicationInfo.uid != realCallingUid)) {
            //如果要启动的Activity与上前正在显示的Activity的用户id（uid）不同，即是不同app，检查是否允许切换App
            if (!mService.checkAppSwitchAllowedLocked(callingPid, callingUid,
                    realCallingPid, realCallingUid, "Activity start")) {
                if (!(restrictedBgActivity && handleBackgroundActivityAbort(r))) {
                    //不允许切换，则把要启动的Activity添加到ActivityStartController的mPendingActivityLaunches中
                    mController.addPendingActivityLaunch(new PendingActivityLaunch(r,
                            sourceRecord, startFlags, stack, callerApp));
                }
                ActivityOptions.abort(checkedOptions);
                //直接返回
                return ActivityManager.START_SWITCHES_CANCELED;
            }
        }
		//上面已经检查了禁切换，所以这里允许切换，一般是用户按下home，或者是用户点击主页的app图标
        mService.onStartActivitySetDidAppSwitch();
        //处理之前加入到ActivityStartController的mPendingActivityLaunches的Activity
        //，该方法最终也是会调用下面的这个startActivity
        mController.doPendingActivityLaunches(false);

        final int res = startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,true /* doResume */, checkedOptions, inTask, outActivity, restrictedBgActivity);
        mSupervisor.getActivityMetricsLogger().notifyActivityLaunched(res, outActivity[0]);
        return res;
    }
```

```java
    private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,ActivityRecord[] outActivity, boolean restrictedBgActivity) {
        int result = START_CANCELED;
        final ActivityStack startedActivityStack;
        try {
            //通知WindowManagerService暂停布局
            mService.mWindowManager.deferSurfaceLayout();
            result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,startFlags, doResume, options, inTask, outActivity,restrictedBgActivity);
        } finally {
            ...
            //启动Activity后，通知WindowManagerService继续布局
            mService.mWindowManager.continueSurfaceLayout();
        }

        postStartActivityProcessing(r, result, startedActivityStack);

        return result;
    }

```

之前的startActivity函数的作用主要就是根据传入的参数，创建要启动的Activity的ActivityRecord，以及处理一些情况。最终交给startActivityUnchecked

#### 2.6 ActivityStarter#startActivityUnchecked

```java
    private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity, boolean restrictedBgActivity) {
        //初始化状态，这里会将ActivityStarter.mStartActivity设置为r，即药启动的Activity
        //设置当前的Activity为mSourceRecord
        setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,voiceInteractor, restrictedBgActivity);

        final int preferredWindowingMode = mLaunchParams.mWindowingMode;
		//计算Launch Flags即启动模式
        computeLaunchingTaskFlags();
		//计算启动的Activity的ActivityStack是哪个
        computeSourceStack();

        mIntent.setFlags(mLaunchFlags);
		//根据flags获取是否存在可以复用的Activity
        ActivityRecord reusedActivity = getReusableIntentActivity();

        ...
       	
        if (reusedActivity != null) {
            ...
            //存在可复用地Activity，则走此逻辑，
            //若启动模式是SingleInSTANCE，会调用onNewIntent
        }
		...
        //要启动地Activity与Task顶部的Activity相同，判断是否继续启动
        final ActivityStack topStack = mRootActivityContainer.getTopDisplayFocusedStack();
        final ActivityRecord topFocused = topStack.getTopActivity();
        final ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(mNotTop);
        final boolean dontStart = top != null && mStartActivity.resultTo == null
                && top.mActivityComponent.equals(mStartActivity.mActivityComponent)
                && top.mUserId == mStartActivity.mUserId
                && top.attachedToProcess()
                && ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
                || isLaunchModeOneOf(LAUNCH_SINGLE_TOP, LAUNCH_SINGLE_TASK))
                && (!top.isActivityTypeHome() || top.getDisplayId() == mPreferredDisplayId);
        if (dontStart) {
            ...
            //重用栈顶的Activity，分发onNewIntent事件
            deliverNewIntent(top);
            return START_DELIVERED_TO_TOP;
        }

                boolean newTask = false;
        final TaskRecord taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
                ? mSourceRecord.getTaskRecord() : null;

        //设置要启动的ActivityRecord的TaskRecord,和ActivityStack
        int result = START_SUCCESS;
        if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
                && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
            //mStartActivity.resultTo表示是哪个Activity启动这个Actvity
            //这里要创建新的TaskRecord
            newTask = true;
            result = setTaskFromReuseOrCreateNewTask(taskToAffiliate);
        } else if (mSourceRecord != null) {
            //如果源Activity存在，则使用这个的TaskRecord
            result = setTaskFromSourceRecord();
        } else if (mInTask != null) {
            result = setTaskFromInTask();
        } else {
            result = setTaskToCurrentTopOrCreateNewTask();
        }
        ...

        mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition,
                mOptions);
        if (mDoResume) {
            final ActivityRecord topTaskActivity =
                    mStartActivity.getTaskRecord().topRunningActivityLocked();
            if (!mTargetStack.isFocusable()
                    || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                    && mStartActivity != topTaskActivity)) {
                mTargetStack.ensureActivitiesVisibleLocked(mStartActivity, 0, !PRESERVE_WINDOWS);
                mTargetStack.getDisplay().mDisplayContent.executeAppTransition();
            } else {
                if (mTargetStack.isFocusable()
                        && !mRootActivityContainer.isTopDisplayFocusedStack(mTargetStack)) {
                    mTargetStack.moveToFront("startActivityUnchecked");
                }
                //调用RootActivityContainer的resumeFocusedStacksTopActivities
                mRootActivityContainer.resumeFocusedStacksTopActivities(
                        mTargetStack, mStartActivity, mOptions);
            }
        } else if (mStartActivity != null) {
            mSupervisor.mRecentTasks.add(mStartActivity.getTaskRecord());
        }
        mRootActivityContainer.updateUserStack(mStartActivity.mUserId, mTargetStack);

        mSupervisor.handleNonResizableTaskIfNeeded(mStartActivity.getTaskRecord(),
                preferredWindowingMode, mPreferredDisplayId, mTargetStack);

        return START_SUCCESS;
    }
```

##### 2.6.1 ActivityStack#startActivityLocked

##### 2.6.2 resumeFocusedStacksTopActivities

然后调用RootActivityContainer.resumeFocusedStacksTopActivities

```java
    boolean resumeFocusedStacksTopActivities(ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

        if (!mStackSupervisor.readyToResume()) {
            return false;
        }

        boolean result = false;
        if (targetStack != null && (targetStack.isTopStackOnDisplay()
                || getTopDisplayFocusedStack() == targetStack)) {
            //调用ActivityStack的resumeTopActivityUncheckedLocked
            result = targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }

       ...

        return result;
    }

```

#### 2.7 ActivityStack#resumeTopActivityUncheckedLocked

```java
    boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mInResumeTopActivity) {
            return false;
        }

        boolean result = false;
        try {
           
            mInResumeTopActivity = true;
            //********************************
            result = resumeTopActivityInnerLocked(prev, options);
            final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
            if (next == null || !next.canTurnScreenOn()) {
                checkReadyForSleep();
            }
        } finally {
            mInResumeTopActivity = false;
        }

        return result;
    }

```



```java
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        if (!mService.isBooting() && !mService.isBooted()) {
            //系统没有进入booting或booted状态，则不允许启动Activity
            return false;
        }
        //这里取到的是ActivityStack的最后一个TaskRecord的最后一个的ActivityRecord，即要启动的Activity的ActivityRecord
		ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
        ...	
		//暂停其他Activity
        boolean pausing = getDisplay().pauseBackStacks(userLeaving, next, false);
        if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
            //2.7.1 当前resumd状态activity不为空，则需要先暂停该Activity
            pausing |= startPausingLocked(userLeaving, false, next, false);
        }
        
          if (pausing && !resumeWhilePausing) {
            //当暂停成功，并且没有设置Activity为FLAG_RESUME_WHILE_PAUSING，并且不是画中画模式
              //进入该分支
           
            ...
            return true;
        } else if (mResumedActivity == next && next.isState(RESUMED)
                && display.allResumedActivitiesComplete()) {
            executeAppTransition(options);
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Top activity resumed (dontWaitForPause) " + next);
            return true;
        }

 		...
        //startPausingLocked中，暂停Activity成功会发送Handler消息，重新进入这个方法
		//下半部分在启动Activity中分析
        return true;
    }


```

resumeTopActivityInnerLocked函数有两个主要功能，一是通知上一个Activity执行onPause，二是启动新的Activity

##### 2.7.1 暂停Activity

这里通过startPausingLocked通知上一个Activity进入onPause

```java
    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
            ActivityRecord resuming, boolean pauseImmediately) {
        ...
        //pre为当前的Activity
        ActivityRecord prev = mResumedActivity;
        ...   
        mPausingActivity = prev;
        mLastPausedActivity = prev;
        mLastNoHistoryActivity = (prev.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_HISTORY) != 0
                || (prev.info.flags & ActivityInfo.FLAG_NO_HISTORY) != 0 ? prev : null;
        //设置当前的Activity状态为PAUSING
        prev.setState(PAUSING, "startPausingLocked");
        prev.getTaskRecord().touchActiveTime();
        clearLaunchTime(prev);

        mService.updateCpuStats();
        if (prev.attachedToProcess()) {
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Enqueueing pending pause: " + prev);
            try {
                EventLogTags.writeAmPauseActivity(prev.mUserId, System.identityHashCode(prev),prev.shortComponentName, "userLeaving=" + userLeaving);
				//执行暂停Activity的事务
                mService.getLifecycleManager().scheduleTransaction(prev.app.getThread(),
                        prev.appToken, PauseActivityItem.obtain(prev.finishing, userLeaving,prev.configChangeFlags, pauseImmediately));
            } catch (Exception e) {
				...
            }
        } else {
			...
        }
    }

```

mService为ATMS对象，mService.getLifecycleManager()得到ATMS的成员变量ClientLifecycleManager对象，并执行scheduleTransaction，传入一个PauseActivityItem，表示将当前RESUME状态的Activity的状态设为PAUSE。

```java
    void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        transaction.schedule();
        if (!(client instanceof Binder)) {
            transaction.recycle();
        }
    }

    void scheduleTransaction(@NonNull IApplicationThread client, @NonNull IBinder activityToken,
            @NonNull ActivityLifecycleItem stateRequest) throws RemoteException {
        //将本地的app进程Binder代理IApplicationThread和actiivtyToken以及stateRequest封装为ClientTransaction对象
        final ClientTransaction clientTransaction = transactionWithState(client, activityToken,
                stateRequest);
        scheduleTransaction(clientTransaction);
    }

```

```java
//ClientTransaction.java 中
	private IApplicationThread mClient;

    public void schedule() throws RemoteException {
        mClient.scheduleTransaction(this);
    }

```

我们看到，最终调用的IApplicationThread的scheduleTransaction方法，这是一个**远程调用**，会调用app进程中ApplicationThread中的`scheduleTransaction`：

```java
//ActivityThread.java#ApplicationThread   
        @Override
        public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
            ActivityThread.this.scheduleTransaction(transaction);
        }
```

ApplicationThread会调用ActivityThread中的同名方法，这个方法是在ActivityThread的父类ClientTransactionHandler中实现的：

```java
    void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }

```

H是一个ActivityThread的内部Handler类，这里直接发送事务交给这个Handler处理：

```java
       public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                ...
                case EXECUTE_TRANSACTION:
                    final ClientTransaction transaction = (ClientTransaction) msg.obj;
                    mTransactionExecutor.execute(transaction);
                    if (isSystem()) {
                        transaction.recycle();
                    }
                    break;
    				...
            }
        }
    }

```

Handler接受到这个消息，又会将消息交给ActivityThread的成员变量mTransactionExecutor处理，它的类型是TransactionExecutor，职责就是处理事务。

```java
//TransactionExecutor.java    
public void execute(ClientTransaction transaction) {
        ...

        executeCallbacks(transaction);

        executeLifecycleState(transaction);
        mPendingActions.clear();
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "End resolving transaction");
    }

   private void executeLifecycleState(ClientTransaction transaction) {
        final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
        ...
        lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
        lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }

```

TransactionExecutor最终将事务交给ActivityLifecycleItem本身去执行，这里的lifecycleItem，根据前文，我们知道是**PauseActivityItem**：

```java
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
        client.handlePauseActivity(token, mFinished, mUserLeaving, mConfigChanges, pendingActions,
                "PAUSE_ACTIVITY_ITEM");
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }

```

PauseActivityItem的execute调用了ClientTransactionHandler#handlePauseActivity，我们知道实际上这个client就是ActivityThread

```java
    @Override
    public void handlePauseActivity(IBinder token, boolean finished, boolean userLeaving,
            int configChanges, PendingTransactionActions pendingActions, String reason) {
        //得到指定Activity（即当前要pause的Activity）的ActivityClientRecord对象
        ActivityClientRecord r = mActivities.get(token);
        if (r != null) {
            ...
            performPauseActivity(r, finished, reason, pendingActions);
            ...
        }
    }

    private Bundle performPauseActivity(ActivityClientRecord r, boolean finished, String reason,
            PendingTransactionActions pendingActions) {
 		...
        performPauseActivityIfNeeded(r, reason);
		...
        return shouldSaveState ? r.state : null;
    }

    private void performPauseActivityIfNeeded(ActivityClientRecord r, String reason) {
        if (r.paused) {
            return;
        }
        reportTopResumedActivityChanged(r, false /* onTop */, "pausing");

        try {
            r.activity.mCalled = false;
            //通过mInstrumentation分发事件
            mInstrumentation.callActivityOnPause(r.activity);
            ..
        } catch (SuperNotCalledException e) {
            throw e;
        } catch (Exception e) {
            ..
        }
        
        r.setState(ON_PAUSE);
    }

```

我们看到这里一路调用，最终调用了Instrumentation#callActivityOnPause：

```java
    //Instrumentation
    public void callActivityOnPause(Activity activity) {
        activity.performPause();
    }
    
    final void performPause() {
        dispatchActivityPrePaused();
        mDoReportFullyDrawn = false;
        mFragments.dispatchPause();
        mCalled = false;
        //调用生命周期方法
        onPause();
        writeEventLog(LOG_AM_ON_PAUSE_CALLED, "performPause");
        mResumed = false;
        if (!mCalled && getApplicationInfo().targetSdkVersion
                >= android.os.Build.VERSION_CODES.GINGERBREAD) {
            throw new SuperNotCalledException(
                    "Activity " + mComponent.toShortString() +
                    " did not call through to super.onPause()");
        }
        dispatchActivityPostPaused();
    }

```

##### 2.7.2 启动新Activity

###### 2.7.2.1 app进程通知ATMS已经暂停Activity

在app进程进行完暂停Activity之后，会调用PauseActivityItem的postExecute方法：

```java
   private void executeLifecycleState(ClientTransaction transaction) {
        final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
        ...
        lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
        //执行postExecute，通知ATMS app进程暂停Activity成功
        lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }
```

```java
    @Override
    public void postExecute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        if (mDontReport) {//mDontReport为PauseActivityItem.obtain的第四个参数，为flase
            return;
        }
        try {
            //调用ATMS的activityPaused方法
            ActivityTaskManager.getService().activityPaused(token);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }

```

###### 2.7.2.2 ATMS通知ActivityStack已经暂停Activity

```java
    @Override
    public final void activityPaused(IBinder token) {
        final long origId = Binder.clearCallingIdentity();
        synchronized (mGlobalLock) {
            //通过Activity.mToken得到ActivityRecord，然后得到ActivityStack
            ActivityStack stack = ActivityRecord.getStackLocked(token);
            if (stack != null) {
                stack.activityPausedLocked(token, false);
            }
        }
        Binder.restoreCallingIdentity(origId);
    }

```

###### 2.7.2.3 ActivityStack进行Activity暂停之后的工作

```java
    final void activityPausedLocked(IBinder token, boolean timeout) {
        if (DEBUG_PAUSE) Slog.v(TAG_PAUSE,
            "Activity paused: token=" + token + ", timeout=" + timeout);

        final ActivityRecord r = isInStackLocked(token);

        if (r != null) {
            //移除暂停Activity超时的Handler的消息
            mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
            if (mPausingActivity == r) {
                //验证ActivityStack中的mPausingActivity与暂停的Activity是否是同一个
                if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to PAUSED: " + r
                        + (timeout ? " (due to timeout)" : " (pause complete)"));
                mService.mWindowManager.deferSurfaceLayout();
                try {
                    completePauseLocked(true /* resumeNext */, null /* resumingActivity */);
                } finally {
                    mService.mWindowManager.continueSurfaceLayout();
                }
                return;
            } else {
				...
            }
        }
        mRootActivityContainer.ensureActivitiesVisible(null, 0, !PRESERVE_WINDOWS);
    }

```

activityPausedLocked首先验证app进程暂停的Activity与ActivityStack中记录的要暂停的Activity，相同则执行`completePauseLocked`

```java
    private void completePauseLocked(boolean resumeNext, ActivityRecord resuming) {
        ActivityRecord prev = mPausingActivity;
        if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Complete pause: " + prev);

        if (prev != null) {
            prev.setWillCloseOrEnterPip(false);
            final boolean wasStopping = prev.isState(STOPPING);
            //设置要暂停的ActivityRecord的State为PAUSED
            prev.setState(PAUSED, "completePausedLocked");
         	...
            //设置ActivityStack的mPausingActivity为null
            mPausingActivity = null;
        }
      
        if (resumeNext) {
            final ActivityStack topStack = mRootActivityContainer.getTopDisplayFocusedStack();
            if (!topStack.shouldSleepOrShutDownActivities()) {
                mRootActivityContainer.resumeFocusedStacksTopActivities(topStack, prev, null);
            } else {
                checkReadyForSleep();
                ActivityRecord top = topStack.topRunningActivityLocked();
                if (top == null || (prev != null && top != prev)) {
                    mRootActivityContainer.resumeFocusedStacksTopActivities();
                }
            }
        }
		...
    }

```

completePauseLocked首先设置mPausingActivity的状态为PAUSED，然后将值设为null。由于这里的方法参数resumeNext为ture，最后又重新来到RootActivityContainer.resumeFocusedStacksTopActivities，在2.7.1暂停Activity一节中，我们知道最终会来到resumeTopActivityInnerLocked，然后执行startPausingLocked开始进行暂停Activity。而这一次重新来到这里，则不会进入startPausingLocked中，具体看下面分析

```java
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        ...
        //由于mResumedActivity为null，pausing为false
        boolean pausing = getDisplay().pauseBackStacks(userLeaving, next, false);
        if (mResumedActivity != null) {
            //mResumedActivity为null，此时不进入
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
            pausing |= startPausingLocked(userLeaving, false, next, false);
        }
        if (pausing && !resumeWhilePausing) {
            ...
            return true;
        } else if (mResumedActivity == next && next.isState(RESUMED)
                && display.allResumedActivitiesComplete()) {
			...
            return true;
        }
       	...
		//代码执行到此处，next指要启动的Activity
        if (next.attachedToProcess()) {
            ...
        } else {
            // 一个新的Activity并没有关联到进程，所以进入此处
            if (!next.hasBeenLaunched) {
                next.hasBeenLaunched = true;
            } else {
                if (SHOW_APP_STARTING_PREVIEW) {
                    next.showStartingWindow(null /* prev */, false /* newTask */,
                            false /* taskSwich */);
                }
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Restarting: " + next);
            }
            if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Restarting " + next);
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
        }

        return true;
    }

```





回到startPausingLocked函数中，让我们看到它他发送暂停Activity的请求之后的操作：

```java
    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
            ActivityRecord resuming, boolean pauseImmediately) {
                if (prev.attachedToProcess()) {
        	...
            try {
                //通过Binder执行暂停Activity的事务
                mService.getLifecycleManager().scheduleTransaction(prev.app.getThread(),
                        prev.appToken, PauseActivityItem.obtain(prev.finishing, userLeaving,
                                prev.configChangeFlags, pauseImmediately));
            } catch (Exception e) {
                Slog.w(TAG, "Exception thrown during pause", e);
                mPausingActivity = null;
                mLastPausedActivity = null;
                mLastNoHistoryActivity = null;
            }
        } else {
            mPausingActivity = null;
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }

        ...
        //事务执行成功，则mPausingActivity不为null
        if (mPausingActivity != null) {  
			...
            //pauseImmediately是方法的参数，resumeTopActivityInnerLocked中传入false
            if (pauseImmediately) {
                completePauseLocked(false, resuming);
                return false;
            } else {
                //进入该分支
                schedulePauseTimeout(prev);
                return true;
            }

        } else {
            // This activity failed to schedule the
            // pause, so just treat it as being paused now.
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Activity not running, resuming next.");
            if (resuming == null) {
                mRootActivityContainer.resumeFocusedStacksTopActivities();
            }
            return false;
        }
    }

```

###### 2.7.2.1 发送Handler

```java
private void schedulePauseTimeout(ActivityRecord r) {
    final Message msg = mHandler.obtainMessage(PAUSE_TIMEOUT_MSG);
    msg.obj = r;
    r.pauseTime = SystemClock.uptimeMillis();
    mHandler.sendMessageDelayed(msg, PAUSE_TIMEOUT);//PAUSE_TIMEOUT=500
    if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Waiting for pause to complete...");
}
```

这里通过mHandler发送了一个PAUSE_TIMEOUT_MSG 的 Message，并且延时了500ms发送，Message带有当前要暂停的ActivityRecord。mHandler对象的类型是ActivityStack的内部类ActivityStackHandler，继承Handler：

```java
  private class ActivityStackHandler extends Handler {
		...
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case PAUSE_TIMEOUT_MSG: {
                    ActivityRecord r = (ActivityRecord)msg.obj;
                    Slog.w(TAG, "Activity pause timeout for " + r);
                    synchronized (mService.mGlobalLock) {
                        if (r.hasProcess()) {
                            mService.logAppTooSlow(r.app, r.pauseTime, "pausing " + r);
                        }
                        activityPausedLocked(r.appToken, true);
                    }
                } break;
                ...
            }
 		}
 }
```











































