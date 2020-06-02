### Android系统启动流程

##### 1.启动电源以及系统启动

​	当电源按下时，应道芯片代码从ROM中执行，加载引导程序BootLoader到RAM中，然后执行。

##### 2.引导程序BootLoader

​	引导程序BootLoader是在Android操作系统开始运行前的一个小程序，它的主要作用是把系统OS拉起来并运行

##### 3. Linux内核启动

​	当内核启动时，设置缓存、被保护存储器、计划列表、加载驱动。当内核完成系统设置时，首先启动Init进程，执行system\core\rootdir\main.cp中的main函数。在main函数中，会进行一些属性的初始化以及对Init.rc文件进行解析。

```c++
//init.rc
import /init.environ.rc
import /system/etc/init/hw/init.usb.rc
import /init.${ro.hardware}.rc
import /vendor/etc/init/hw/init.${ro.hardware}.rc
import /system/etc/init/hw/init.usb.configfs.rc
import /system/etc/init/hw/init.${ro.zygote}.rc
```

init.rc中导入.rc文件来初始化相应模块。对于android10来说，它提供了多个zygote相关的.rc文件，有init.zygote32.rc、init.zygote64.rc等，我们看到 Init.zygote64.rc中的实现：

```c++
//init.zygote64.rc
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote stream 660 root system
    socket usap_pool_primary stream 660 root system
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    writepid /dev/cpuset/foreground/tasks
```

由第一行，我们可以知道它首先会执行`/system/bin/app_process64`，后面的是它的参数。

app_process64的实现是app_main.cpp，它位于`frameworks\base\cmds\app_process`，在该文件夹下面还有它的配置文件Android.bp，根据该配置文件，我们可以知道，app_process的确对应着app_main.cpp

```c++
//Android.bp
cc_binary {
    name: "app_process",
    srcs: ["app_main.cpp"],
	//......
}
```

所以，这里会进入到app_main中的main函数中，并传入参数`-Xzygote /system/bin --zygote --start-system-server`

##### 4. Init进程

上一节我们了解到init进程会进入到app_main.cpp的main函数中，即下面的代码：

```c++
//app_main.cpp
int main(int argc, char* const argv[])
{
    //......
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
	
    //处理参数，在第一个无法识别处停止
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;

    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }
    // 定义参数args
	Vector<String8> args;
    if (!className.isEmpty()) {
      
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);

        if (!LOG_NDEBUG) {
          String8 restOfArgs;
          char* const* argv_new = argv + i;
          int argc_new = argc - i;
          for (int k = 0; k < argc_new; ++k) {
            restOfArgs.append("\"");
            restOfArgs.append(argv_new[k]);
            restOfArgs.append("\" ");
          }
          ALOGV("Class name = %s, args = %s", className.string(), restOfArgs.string());
        }
    } else {
        // We're in zygote mode.
        maybeCreateDalvikCache();

        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }

        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }

        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);

        // In zygote mode, pass all remaining arguments to the zygote
        // main() method.
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }
    if (zygote) {
		//通过AppRunTime.start函数，调用ZygoteInit的main方法，进入Java世界
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}
```

在app_main.cpp的main函数中，会构造一个`AppRuntime`对象，在函数的最后，调用它的`start()`函数，根据我们在init.zygote64.rc中，看到的参数包含有--zygote，zygote为true，所以启动的是ZygoteInit类。

AppRuntime定义在app_main.cpp内，但是它是AndroidRuntime的派生类，AppRuntime主要实现了一些回调函数，并且并没有重写start()函数

```c++
//frameworks\base\core\jni\AndroidRuntime.cpp
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    /* 1. 首先启动虚拟机 */
    JniInvocation jni_invocation;
    // 1.1通过JniInvocation初始化一些本地方法函数
    jni_invocation.Init(NULL);
    JNIEnv* env;
	// 1.2启动虚拟机
    if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) {
        return;
    }
    // 1.3虚拟机创建成功的回调函数
    onVmCreated(env);

    /*
     * 2. 在startReg(env)中，注册Android的JNI函数
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

    /*
     * 3. 解析要启动的Java类的类名以及参数
     */
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;
	//3.1 得到一个String数组，长度为 options的长度加1
    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    //将className即com.android.internal.os.ZygoteInit放入String数组的第一个
    env->SetObjectArrayElement(strArray, 0, classNameStr);
	//将剩下的options放入String数组
    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }
	//3.2 找到ZygoteInit这个类，并执行类中的main()
	//3.2.1 首先将 传入的com.android.internal.os.ZygoteInit 将.替换为com/android/internal/os/ZygoteInit
    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    //3.2.2找到ZygoteInit这个类
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
    } else {
        //3.2.3 找到ZygoteInit类后，找类中的main()方法
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
        } else {
        	//3.2.4 找到方法后，调用该方法，并传入参数strArray
            env->CallStaticVoidMethod(startClass, startMeth, strArray);

#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
    free(slashClassName);
    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
}
```



##### 5.Zygote进程

ZygoteInit.java中的main函数是Zygote进程的入口，在这里会启动Zygote服务，加载资源，并且为fork application处理相关的任务和准备进程

```java
    public static void main(String argv[]) {
        ZygoteServer zygoteServer = null;

        // Mark zygote start. This ensures that thread creation will throw
        // an error.
        ZygoteHooks.startZygoteNoThreadCreation();

        // Zygote goes into its own process group.
        try {
            Os.setpgid(0, 0);
        } catch (ErrnoException ex) {
            throw new RuntimeException("Failed to setpgid(0,0)", ex);
        }

        Runnable caller;
        try {
            // Report Zygote start time to tron unless it is a runtime restart
            if (!"1".equals(SystemProperties.get("sys.boot_completed"))) {
                MetricsLogger.histogram(null, "boot_zygote_init",
                        (int) SystemClock.elapsedRealtime());
            }

            String bootTimeTag = Process.is64Bit() ? "Zygote64Timing" : "Zygote32Timing";
            TimingsTraceLog bootTimingsTraceLog = new TimingsTraceLog(bootTimeTag,
                    Trace.TRACE_TAG_DALVIK);
            bootTimingsTraceLog.traceBegin("ZygoteInit");
            RuntimeInit.preForkInit();

            boolean startSystemServer = false;
            String zygoteSocketName = "zygote";
            String abiList = null;
            boolean enableLazyPreload = false;
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if ("--enable-lazy-preload".equals(argv[i])) {
                    enableLazyPreload = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    zygoteSocketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            final boolean isPrimaryZygote = zygoteSocketName.equals(Zygote.PRIMARY_SOCKET_NAME);

            if (abiList == null) {
                throw new RuntimeException("No ABI list supplied.");
            }

            // In some configurations, we avoid preloading resources and classes eagerly.
            // In such cases, we will preload things prior to our first fork.
            if (!enableLazyPreload) {
                bootTimingsTraceLog.traceBegin("ZygotePreload");
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                        SystemClock.uptimeMillis());
                preload(bootTimingsTraceLog);
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                        SystemClock.uptimeMillis());
                bootTimingsTraceLog.traceEnd(); // ZygotePreload
            }

            // Do an initial gc to clean up after startup
            bootTimingsTraceLog.traceBegin("PostZygoteInitGC");
            gcAndFinalize();
            bootTimingsTraceLog.traceEnd(); // PostZygoteInitGC

            bootTimingsTraceLog.traceEnd(); // ZygoteInit

            Zygote.initNativeState(isPrimaryZygote);

            ZygoteHooks.stopZygoteNoThreadCreation();

            zygoteServer = new ZygoteServer(isPrimaryZygote);

            if (startSystemServer) {
                Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

                // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
                // child (system_server) process.
                if (r != null) {
                    r.run();
                    return;
                }
            }

            Log.i(TAG, "Accepting command socket connections");

            // The select loop returns early in the child process after a fork and
            // loops forever in the zygote.
            caller = zygoteServer.runSelectLoop(abiList);
        } catch (Throwable ex) {
            Log.e(TAG, "System zygote died with exception", ex);
            throw ex;
        } finally {
            if (zygoteServer != null) {
                zygoteServer.closeServerSocket();
            }
        }

        // We're in the child process and have exited the select loop. Proceed to execute the
        // command.
        if (caller != null) {
            caller.run();
        }
    }
```

