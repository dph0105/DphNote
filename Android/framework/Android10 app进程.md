### 1.App进程启动

```java

public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");

        // 安装选择性系统调用拦截
        AndroidOs.install();
		...
		//初始化app主线程Looper
        Looper.prepareMainLooper();

        //得到传入的序列号 传入的参数如果有 seq=114 这样子的，startSeq就是114
        long startSeq = 0;
        if (args != null) {
            for (int i = args.length - 1; i >= 0; --i) {
                if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                    startSeq = Long.parseLong(
                            args[i].substring(PROC_START_SEQ_IDENT.length()));
                }
            }
        }
        //创建ActivityThread对象
        ActivityThread thread = new ActivityThread();
        //调用ActivityThread.attach 传入参数false，表示是应用程序进程
        thread.attach(false, startSeq);
		
        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
		...

      	//开启消息循环
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }

```

#### 1.1 进入ActivityThread.attach

```java
    private void attach(boolean system, long startSeq) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            //在ActivityThread.main方法中，我们知道传入的system为flase，进入该分支
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            //获取AMS的本地代理类
            final IActivityManager mgr = ActivityManager.getService();
            try {
                //通过Binder调用远端的AMS的attachApplication方法
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            // Watch for getting close to heap limit.
            BinderInternal.addGcWatcher(new Runnable() {
                @Override public void run() {
                    if (!mSomeActivitiesChanged) {
                        return;
                    }
                    Runtime runtime = Runtime.getRuntime();
                    long dalvikMax = runtime.maxMemory();
                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                    if (dalvikUsed > ((3*dalvikMax)/4)) {
                        if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
                                + " total=" + (runtime.totalMemory()/1024)
                                + " used=" + (dalvikUsed/1024));
                        mSomeActivitiesChanged = false;
                        try {
                            ActivityTaskManager.getService().releaseSomeActivities(mAppThread);
                        } catch (RemoteException e) {
                            throw e.rethrowFromSystemServer();
                        }
                    }
                }
            });
        } else {
            //system_server进程会进入该分支
        }

        ViewRootImpl.ConfigChangedCallback configChangedCallback
                = (Configuration globalConfig) -> {
            synchronized (mResourcesManager) {
                // We need to apply this change to the resources immediately, because upon returning
                // the view hierarchy will be informed about it.
                if (mResourcesManager.applyConfigurationToResourcesLocked(globalConfig,
                        null /* compat */)) {
                    updateLocaleListFromAppContext(mInitialApplication.getApplicationContext(),
                            mResourcesManager.getConfiguration().getLocales());

                    // This actually changed the resources! Tell everyone about it.
                    if (mPendingConfiguration == null
                            || mPendingConfiguration.isOtherSeqNewer(globalConfig)) {
                        mPendingConfiguration = globalConfig;
                        sendMessage(H.CONFIGURATION_CHANGED, globalConfig);
                    }
                }
            }
        };
        ViewRootImpl.addConfigCallback(configChangedCallback);
    }

```

##### 1.1.1 得到ActivityManagerService本地Binder

在attach中，如果传入的system参数为false，则表示是app进程，进入相应分支。首先会通过`ActivityManager.getService()`得到ActivityManagerService的本地Binder代理。

```java
    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    @UnsupportedAppUsage
    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
```

##### 1.1.2 调用ActivityManagerService的attachApplication方法

通过ActivityManagerService的Binder，调用它的attachApplication方法，执行该方法的进程是启动和维护ActivityManagerService的SystemServer进程。

并且我们看到，这里传入的参数为mAppThread，这是一个ApplicationThread类，代表着app进程中的Binder。

```java
  private class ApplicationThread extends IApplicationThread.Stub {
        private static final String DB_INFO_FORMAT = "  %8s %8s %14s %14s  %s";
	...
  }

```

当我们想要访问systemserver中的系统服务时（这里为ActivityManagerService），由于ServiceManager进程作为Service时，它在Binder驱动中的句柄为0，任何Client想要访问ServiceManager，可以直接通过句柄0来进行访问，从而得到AMS的远程代理Binder。因此，从app进程到AMS的消息路线已经打通，但是AMS传递消息到app进程也是一个跨进程通信，需要得到app进程的binder。因此这里直接将ApplicationThread传递到AMS中，用于通信，我们接下来看，是否是这样。

### 2. 来到SystemServer进程中

```java
   //frameworks\base\services\core\java\com\android\server\am\ActivityManagerService.java
	@Override
    public final void attachApplication(IApplicationThread thread, long startSeq) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid, callingUid, startSeq);
            Binder.restoreCallingIdentity(origId);
        }
    }

```



```java
   
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
        //1.获得ProcessRecord
        ProcessRecord app;
        long startTime = SystemClock.uptimeMillis();
        long bindApplicationTimeMillis;
        if (pid != MY_PID && pid >= 0) {
            synchronized (mPidsSelfLocked) {
                app = mPidsSelfLocked.get(pid);
            }
            if (app != null && (app.startUid != callingUid || app.startSeq != startSeq)) {
                String processName = null;
                final ProcessRecord pending = mProcessList.mPendingStarts.get(startSeq);
                if (pending != null) {
                    processName = pending.processName;
                }
                final String msg = "attachApplicationLocked process:" + processName
                        + " startSeq:" + startSeq
                        + " pid:" + pid
                        + " belongs to another existing app:" + app.processName
                        + " startSeq:" + app.startSeq;
                Slog.wtf(TAG, msg);
                // SafetyNet logging for b/131105245.
                EventLog.writeEvent(0x534e4554, "131105245", app.startUid, msg);
                // If there is already an app occupying that pid that hasn't been cleaned up
                cleanUpApplicationRecordLocked(app, false, false, -1,
                            true /*replacingPid*/);
                mPidsSelfLocked.remove(app);
                app = null;
            }
        } else {
            app = null;
        }

        // It's possible that process called attachApplication before we got a chance to
        // update the internal state.
        if (app == null && startSeq > 0) {
            final ProcessRecord pending = mProcessList.mPendingStarts.get(startSeq);
            if (pending != null && pending.startUid == callingUid && pending.startSeq == startSeq
                    && mProcessList.handleProcessStartedLocked(pending, pid, pending
                            .isUsingWrapper(),
                            startSeq, true)) {
                app = pending;
            }
        }


        try {
            ...
            bindApplicationTimeMillis = SystemClock.elapsedRealtime();
            mAtmInternal.preBindApplication(app.getWindowProcessController());
            final ActiveInstrumentation instr2 = app.getActiveInstrumentation();
            if (mPlatformCompat != null) {
                mPlatformCompat.resetReporting(app.info);
            }
            //调用ApplicationThread的runIsolatedEntryPoint或者bindApplication
            if (app.isolatedEntryPoint != null) {
                thread.runIsolatedEntryPoint(app.isolatedEntryPoint, app.isolatedEntryPointArgs);
            } else if (instr2 != null) {
                thread.bindApplication(processName, appInfo, providers,
                        instr2.mClass,
                        profilerInfo, instr2.mArguments,
                        instr2.mWatcher,
                        instr2.mUiAutomationConnection, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.isPersistent(),
                        new Configuration(app.getWindowProcessController().getConfiguration()),
                        app.compat, getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, autofillOptions, contentCaptureOptions,
                        app.mDisabledCompatChanges);
            } else {
                thread.bindApplication(processName, appInfo, providers, null, profilerInfo,
                        null, null, null, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.isPersistent(),
                        new Configuration(app.getWindowProcessController().getConfiguration()),
                        app.compat, getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, autofillOptions, contentCaptureOptions,
                        app.mDisabledCompatChanges);
            }
            if (profilerInfo != null) {
                profilerInfo.closeFd();
                profilerInfo = null;
            }
			//ProcessRecord记录下ApplicationThread
            app.makeActive(thread, mProcessStats);
            checkTime(startTime, "attachApplicationLocked: immediately after bindApplication");
            mProcessList.updateLruProcessLocked(app, false, null);
            checkTime(startTime, "attachApplicationLocked: after updateLruProcessLocked");
            app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
        } catch (Exception e) {
            ...
        }

        ...
        return true;
    }
```

attachApplicationLocked方法比较长，主要有三部分

#### 2.1 获得ProcessRecord

首先获取ProcessRecord，ProcessRecord的用于描述进程的数据结构，例如：

1. processName：记录进程名，默认情况下进程名和该进程运行的第一个apk的包名是相同的，当然也可以自定义进程名；
2. pid: 记录进程pid，该值在由进程创建时内核所分配的。
3. thread：执行完attachApplicationLocked()方法，会把客户端进程ApplicationThread的binder服务的代理端传递到 AMS，并保持到ProcessRecord的成员变量thread；
4. info：记录运行在该进程的第一个应用
5. pkgList: 记录运行在该进程中所有的包名，比如通过addPackage()添加
6.  ...

#### 2.2 调用ApplicationThread.bindApplication

```java
        public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableBinderTracking, boolean trackAllocation,
                boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
                String buildSerial, AutofillOptions autofillOptions,
                ContentCaptureOptions contentCaptureOptions, long[] disabledCompatChanges) {
            if (services != null) {
                // Setup the service cache in the ServiceManager
                ServiceManager.initServiceCache(services);
            }

            setCoreSettings(coreSettings);

            AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            data.debugMode = debugMode;
            data.enableBinderTracking = enableBinderTracking;
            data.trackAllocation = trackAllocation;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;
            data.buildSerial = buildSerial;
            data.autofillOptions = autofillOptions;
            data.contentCaptureOptions = contentCaptureOptions;
            data.disabledCompatChanges = disabledCompatChanges;
            sendMessage(H.BIND_APPLICATION, data);
        }
```

这里构造了一个AppBindData对象，然后通过Handler发送消息

```java
 public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                 ...
             }
   }
```

handler接收到消息后，执行handleBindApplication方法

```java

 private void handleBindApplication(AppBindData data) {
   //通过反射创建Instrumentation
   try {
       final ClassLoader cl = instrContext.getClassLoader();
       mInstrumentation = (Instrumentation)
       cl.loadClass(data.instrumentationName.getClassName()).newInstance();
    }
	...
     
 	//获得LoadedApk，表示当前加载的apk
   data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
 
   ......
  
   try {
       //创建一个Application对象     
       Application app = data.info.makeApplication(data.restrictedBackupMode,null);
       mInitialApplication = app;
 
       
       try {
          //通过mInstrumentation对象，回调Application的onCreate()方法  
           mInstrumentation.callApplicationOnCreate(app);
       } catch (Exception e) {
            ......
        } finally {
            ......
        }
 
        ......
}
```

##### 2.2.1 创建Application对象

getPackageInfoNoCheck(data.appInfo, data.compatInfo)方法会得到一个LoadedApk对象，赋值给data.info。LoadedApk表示一个当前加载的apk，通过它的makeApplication，创建Application对象

```java
    public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {
            return mApplication;
        }
		...

        Application app = null;

        String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }

        try {
            java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                        "initializeJavaContextClassLoader");
                initializeJavaContextClassLoader();
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }
            //创建ContextImpl实例
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            //创建Application实例
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {
            if (!mActivityThread.mInstrumentation.onException(app, e)) {
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                throw new RuntimeException(
                    "Unable to instantiate application " + appClass
                    + ": " + e.toString(), e);
            }
        }
        mActivityThread.mAllApplications.add(app);
        mApplication = app;

        if (instrumentation != null) {
            //这里不会执行，因为传递来的instrumentation为null
            ...
        }

   		...
        return app;
    }
```



```java
    public Application newApplication(ClassLoader cl, String className, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        Application app = getFactory(context.getPackageName())
                .instantiateApplication(cl, className);
        app.attach(context);
        return app;
    }
```

getFactory得到一个AppComponentFactory对象，然后调用它的instantiateApplication方法创建Application对象，默认的AppComponentFactory的实现为：

```java
    public @NonNull Service instantiateService(@NonNull ClassLoader cl,
            @NonNull String className, @Nullable Intent intent)
            throws InstantiationException, IllegalAccessException, ClassNotFoundException {
        return (Service) cl.loadClass(className).newInstance();
    }

```

直接通过反射创建实例。然后调用Application.attach方法，即Application中的attachBaseContext(context);

##### 2.2.2 调用Application的onCreate

回到ActivityThread中的handleBindApplication方法中，在得到Application后，会通过mInstrumentation.callApplicationOnCreate(app)回调Application的onCreate方法：

```
    public void callApplicationOnCreate(Application app) {
        app.onCreate();
    }
```

该方法直接调用app.onCreate();

#### 2.3  ProcessRecord存储ApplicationThread

在ActivityManagerService中，最终还是绕回到app进程中，执行创建Application等一系列工作。而ApplicationThread将会被ProcessRecord存下来。

```java
 public void makeActive(IApplicationThread _thread, ProcessStatsService tracker) {
        if (thread == null) {
            final ProcessState origBase = baseProcessTracker;
            if (origBase != null) {
                origBase.setState(ProcessStats.STATE_NOTHING,
                        tracker.getMemFactorLocked(), SystemClock.uptimeMillis(), pkgList.mPkgList);
                for (int ipkg = pkgList.size() - 1; ipkg >= 0; ipkg--) {
                    StatsLog.write(StatsLog.PROCESS_STATE_CHANGED,
                            uid, processName, pkgList.keyAt(ipkg),
                            ActivityManager.processStateAmToProto(ProcessStats.STATE_NOTHING),
                            pkgList.valueAt(ipkg).appVersion);
                }
                origBase.makeInactive();
            }
            baseProcessTracker = tracker.getProcessStateLocked(info.packageName, info.uid,
                    info.longVersionCode, processName);
            baseProcessTracker.makeActive();
            for (int i=0; i<pkgList.size(); i++) {
                ProcessStats.ProcessStateHolder holder = pkgList.valueAt(i);
                if (holder.state != null && holder.state != origBase) {
                    holder.state.makeInactive();
                }
                tracker.updateProcessStateHolderLocked(holder, pkgList.keyAt(i), info.uid,
                        info.longVersionCode, processName);
                if (holder.state != baseProcessTracker) {
                    holder.state.makeActive();
                }
            }
        }
        thread = _thread;
        mWindowProcessController.setThread(thread);
    }

```

ProcessRecord中的thread字段，记录着app进程作为远程服务端时提供的Binder代理对象

#### 2.4 启动Activity

在AMS的attachApplicationLocked方法中，在创建Application并且调用它的生命周期方法，以及记录下app进程的ApplicationThread后，接下来会做什么呢？

我们看到attachApplicationLocked下面的代码：

```java
 private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
  
  // See if the top visible activity is waiting to run in this process...
        if (normalMode) {
            try {
                didSomething = mAtmInternal.attachApplication(app.getWindowProcessController());
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
                badApp = true;
            }
        }

        // Find any services that should be running in this process...
        if (!badApp) {
            try {
                didSomething |= mServices.attachApplicationLocked(app, processName);
                checkTime(startTime, "attachApplicationLocked: after mServices.attachApplicationLocked");
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown starting services in " + app, e);
                badApp = true;
            }
        }

        // Check if a next-broadcast receiver is in this process...
        if (!badApp && isPendingBroadcastProcessLocked(pid)) {
            try {
                didSomething |= sendPendingBroadcastsLocked(app);
                checkTime(startTime, "attachApplicationLocked: after sendPendingBroadcastsLocked");
            } catch (Exception e) {
                // If the app died trying to launch the receiver we declare it 'bad'
                Slog.wtf(TAG, "Exception thrown dispatching broadcasts in " + app, e);
                badApp = true;
            }
        }

        // Check whether the next backup agent is in this process...
        if (!badApp && backupTarget != null && backupTarget.app == app) {
            if (DEBUG_BACKUP) Slog.v(TAG_BACKUP,
                    "New app is backup target, launching agent for " + app);
            notifyPackageUse(backupTarget.appInfo.packageName,
                             PackageManager.NOTIFY_PACKAGE_USE_BACKUP);
            try {
                thread.scheduleCreateBackupAgent(backupTarget.appInfo,
                        compatibilityInfoForPackage(backupTarget.appInfo),
                        backupTarget.backupMode, backupTarget.userId);
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown creating backup agent in " + app, e);
                badApp = true;
            }
        }

        if (badApp) {
            app.kill("error during init", true);
            handleAppDiedLocked(app, false, true);
            return false;
        }

        if (!didSomething) {
            updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_PROCESS_BEGIN);
            checkTime(startTime, "attachApplicationLocked: after updateOomAdjLocked");
        }

        return true;
    }
```



##### 2.4.1 mAtmInternal.attachApplication

mAtmInternal的类型是ActivityTaskManagerInternal，它是一个抽象类，具体的实现是ActivityTaskManagerService.LocalService。这里首先调用了它的attachApplication方法：

```java
        public boolean attachApplication(WindowProcessController wpc) throws RemoteException {
            synchronized (mGlobalLockWithoutBoost) {
                return mRootActivityContainer.attachApplication(wpc);
            }
        }
```



