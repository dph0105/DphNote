### Android系统启动流程

#### 1.启动电源以及系统启动

​	当电源按下时，应道芯片代码从ROM中执行，加载引导程序BootLoader到RAM中，然后执行。

#### 2.引导程序BootLoader

​	引导程序BootLoader是在Android操作系统开始运行前的一个小程序，它的主要作用是把系统OS拉起来并运行

#### 3.Linux内核启动

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

#### 4. Init进程

上一节我们了解到init进程会进入到app_main.cpp的main函数中，即下面的代码：

```c++
//app_main.cpp
int main(int argc, char* const argv[])
{
    //传到的参数argv为“-Xzygote /system/bin --zygote --start-system-server”
    //......
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
	
    //处理参数，在第一个无法识别处停止
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;

    ++i;
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            //niceName： 64位系统——》zygote64,32位系统——》zygote
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
        //进入zygote模式
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
        	//3.2.4 找到方法后，调用该方法，并传入参数strArray ，即调用ZygoteInit.main()
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

##### #### 5.Zygote进程

ZygoteInit.java中的main函数是Zygote进程的入口，在这里会启动Zygote服务，加载资源，并且为fork application处理相关的任务和准备进程

```java
    public static void main(String argv[]) {
        ZygoteServer zygoteServer = null;

        Runnable caller;
        try {
            
            if (!enableLazyPreload) {
                //... 5.1 如果没有开启懒加载，那么就预加载类和资源
                preload(bootTimingsTraceLog);
                //...
            }

            Zygote.initNativeState(isPrimaryZygote);

            ZygoteHooks.stopZygoteNoThreadCreation();
			//5.2 创建ZygoteServer并创建Socket
            zygoteServer = new ZygoteServer(isPrimaryZygote);

            if (startSystemServer) {
                //5.3 fork出SystemServer进程，并执行
                Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);
                //当r！=null时，表示处于子进程system_server中，运行runnable
                if (r != null) {
                    r.run();
                    return;
                }
            }
            //5.4  能执行到这一步，表示在父进程zygote进程中，进入循环，等待接收消息
            caller = zygoteServer.runSelectLoop(abiList);
        } catch (Throwable ex) {
            Log.e(TAG, "System zygote died with exception", ex);
            throw ex;
        } finally {
            if (zygoteServer != null) {
                zygoteServer.closeServerSocket();
            }
        }

        if (caller != null) {
            caller.run();
        }
    }
```

##### 5.1 preload 预加载

首先看到 第一点preload(bootTimingsTraceLog);

```java
//ZygoteInit.java
static void preload(TimingsTraceLog bootTimingsTraceLog) {
	 beginPreload();
	 //预加载/system/etc/preloaded-classes中的类
	 preloadClasses();
	 //预加载一些应用需要使用但是不能放入bootclasspath中的jar包
	 cacheNonBootClasspathClassLoaders();
	 //预加载资源，包含drawable和color资源
	 preloadResources();
	 nativePreloadAppProcessHALs();
	 //预加载OpenGL或者Vulkan Vulkan也是一个2D、3D绘图接口
	 maybePreloadGraphicsDriver();
	 //通过System.loadLibrary 加载Jni库
	 preloadSharedLibraries();
	 //预加载文本连接符资源
	 preloadTextResources();
	 //预先在Zygote中做好加载WebView的准备
	 WebViewFactory.prepareWebViewInZygote();
	 endPreload();
	 sPreloadComplete = true;
}
```

##### 5.2 创建ZygoteServer并开启Sokcet

在预加载资源后，ZygoteInit会创建ZygoteServer对象，在ZygoteServer的构造方法中，会创建Socket用于接收消息：

```java
    ZygoteServer(boolean isPrimaryZygote) {
        mUsapPoolEventFD = Zygote.getUsapPoolEventFD();

        if (isPrimaryZygote) {
            //使用ZygoteSocket记录Socket
            mZygoteSocket = Zygote.createManagedSocketFromInitSocket(Zygote.PRIMARY_SOCKET_NAME);
            mUsapPoolSocket =
                    Zygote.createManagedSocketFromInitSocket(
                            Zygote.USAP_POOL_PRIMARY_SOCKET_NAME);
        } else {
            mZygoteSocket = Zygote.createManagedSocketFromInitSocket(Zygote.SECONDARY_SOCKET_NAME);
            mUsapPoolSocket =
                    Zygote.createManagedSocketFromInitSocket(
                            Zygote.USAP_POOL_SECONDARY_SOCKET_NAME);
        }

        mUsapPoolSupported = true;
        fetchUsapPoolPolicyProps();
    }
    
    static LocalServerSocket createManagedSocketFromInitSocket(String socketName) {
        int fileDesc;
        final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;

        try {
            String env = System.getenv(fullSocketName);
            fileDesc = Integer.parseInt(env);
        } catch (RuntimeException ex) {
            throw new RuntimeException("Socket unset or invalid: " + fullSocketName, ex);
        }

        try {
            FileDescriptor fd = new FileDescriptor();
            fd.setInt$(fileDesc);//设置文件描述符
            return new LocalServerSocket(fd);//创建Socket的本地服务端
        } catch (IOException ex) {
            ...
        }
    }

```

##### 5.3 创建SystemServer进程

创建完ZygoteServer后，ZygoteInit创建SystemServer进程，我们来看forkSystemServer:

```java
   private static Runnable forkSystemServer(String abiList, String socketName,
            ZygoteServer zygoteServer) {
       //......
       //用args数组保存SystemServer的启动参数
        String args[] = {
                "--setuid=1000",
                "--setgid=1000",
                "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,"
                        + "1024,1032,1065,3001,3002,3003,3006,3007,3009,3010,3011",
                "--capabilities=" + capabilities + "," + capabilities,
                "--nice-name=system_server",
                "--runtime-args",
                "--target-sdk-version=" + VMRuntime.SDK_VERSION_CUR_DEVELOPMENT,
                "com.android.server.SystemServer",
        };
        ZygoteArguments parsedArgs = null;

        int pid;

        try {
            //解析参数
            parsedArgs = new ZygoteArguments(args);
            Zygote.applyDebuggerSystemProperty(parsedArgs);
            Zygote.applyInvokeWithSystemProperty(parsedArgs);
			//fork 子进程，用于运行system_server
            pid = Zygote.forkSystemServer(
                    parsedArgs.mUid, parsedArgs.mGid,
                    parsedArgs.mGids,
                    parsedArgs.mRuntimeFlags,
                    null,
                    parsedArgs.mPermittedCapabilities,
                    parsedArgs.mEffectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }
        if (pid == 0) {
            //当pic==0时，表示在子进程system_server中
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }
            //子进程不需要开启ServerSocket
            zygoteServer.closeServerSocket();
            //完成system_server进程中的工作
            return handleSystemServerProcess(parsedArgs);
        }
        return null;
    }

```

`forkSystemServer()`函数准备参数并且fork出新进程，从它的参数可以看出，system_server进程参数信息uid=1000，gid=1000，进程名为system_server。Linux通过fork创建新进程会有两次返回，当pid == 0时，表示处于新进程中，我们可以看到上面的代码中，此时就会继续处理system_server剩余的工作。当pid>0时，pid表示子进程的id，此时处于父进程中，即Zygote进程中，此时返回null。

##### 5.4 开启Zygote进程中消息循环

通过5.3小节的分析，我们知道当返回null时，表示此时在Zygote进程自身中，那么将会执行`caller = zygoteServer.runSelectLoop(abiList);`

```java
    Runnable runSelectLoop(String abiList) {
        ArrayList<FileDescriptor> socketFDs = new ArrayList<>();
        ArrayList<ZygoteConnection> peers = new ArrayList<>();
		//mZygoteSocket即在5.2节中创建的LocalServerSocket
        socketFDs.add(mZygoteSocket.getFileDescriptor());
        peers.add(null);

        mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;

        while (true) {
            ...
            int pollReturnValue;
            try {
                //处理轮询状态，当pollFds有事件到来，则往下执行，否则阻塞
                pollReturnValue = Os.poll(pollFDs, pollTimeoutMs);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }

            if (pollReturnValue == 0) {
          mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;
                mUsapPoolRefillAction = UsapPoolRefillAction.DELAYED;

            } else {
                boolean usapPoolFDRead = false;
				//
                while (--pollIndex >= 0) {
                    //采用I/O多路复用，没有消息，直接continue，有消息则往下执行
                    if ((pollFDs[pollIndex].revents & POLLIN) == 0) {
                        continue;
                    }

                    if (pollIndex == 0) {
						//pollIndex为0，表示是ZygoteSocket
                        ZygoteConnection newPeer = acceptCommandPeer(abiList);
                        peers.add(newPeer);
                        //添加到socketFDs中
                        socketFDs.add(newPeer.getFileDescriptor());

                    } else if (pollIndex < usapPoolEventFDIndex) {
                        // Session socket accepted from the Zygote server socket

                        try {
                            ZygoteConnection connection = peers.get(pollIndex);
                            final Runnable command = connection.processOneCommand(this);
                            if (mIsForkChild) {
                                // We're in the child. We should always have a command to run at
                                // this stage if processOneCommand hasn't called "exec".
                                if (command == null) {
                                    throw new IllegalStateException("command == null");
                                }

                                return command;
                            } else {
                                // We're in the server - we should never have any commands to run.
                                if (command != null) {
                                    throw new IllegalStateException("command != null");
                                }

                                // We don't know whether the remote side of the socket was closed or
                                // not until we attempt to read from it from processOneCommand. This
                                // shows up as a regular POLLIN event in our regular processing
                                // loop.
                                if (connection.isClosedByPeer()) {
                                    connection.closeSocket();
                                    peers.remove(pollIndex);
                                    socketFDs.remove(pollIndex);
                                }
                            }
                        } catch (Exception e) {
                            if (!mIsForkChild) {
                                // We're in the server so any exception here is one that has taken
                                // place pre-fork while processing commands or reading / writing
                                // from the control socket. Make a loud noise about any such
                                // exceptions so that we know exactly what failed and why.

                                Slog.e(TAG, "Exception executing zygote command: ", e);

                                // Make sure the socket is closed so that the other end knows
                                // immediately that something has gone wrong and doesn't time out
                                // waiting for a response.
                                ZygoteConnection conn = peers.remove(pollIndex);
                                conn.closeSocket();

                                socketFDs.remove(pollIndex);
                            } else {
                                // We're in the child so any exception caught here has happened post
                                // fork and before we execute ActivityThread.main (or any other
                                // main() method). Log the details of the exception and bring down
                                // the process.
                                Log.e(TAG, "Caught post-fork exception in child process.", e);
                                throw e;
                            }
                        } finally {
                            // Reset the child flag, in the event that the child process is a child-
                            // zygote. The flag will not be consulted this loop pass after the
                            // Runnable is returned.
                            mIsForkChild = false;
                        }

                    } else {
                        // Either the USAP pool event FD or a USAP reporting pipe.

                        // If this is the event FD the payload will be the number of USAPs removed.
                        // If this is a reporting pipe FD the payload will be the PID of the USAP
                        // that was just specialized.  The `continue` statements below ensure that
                        // the messagePayload will always be valid if we complete the try block
                        // without an exception.
                        long messagePayload;

                        try {
                            byte[] buffer = new byte[Zygote.USAP_MANAGEMENT_MESSAGE_BYTES];
                            int readBytes =
                                    Os.read(pollFDs[pollIndex].fd, buffer, 0, buffer.length);

                            if (readBytes == Zygote.USAP_MANAGEMENT_MESSAGE_BYTES) {
                                DataInputStream inputStream =
                                        new DataInputStream(new ByteArrayInputStream(buffer));

                                messagePayload = inputStream.readLong();
                            } else {
                                Log.e(TAG, "Incomplete read from USAP management FD of size "
                                        + readBytes);
                                continue;
                            }
                        } catch (Exception ex) {
                            if (pollIndex == usapPoolEventFDIndex) {
                                Log.e(TAG, "Failed to read from USAP pool event FD: "
                                        + ex.getMessage());
                            } else {
                                Log.e(TAG, "Failed to read from USAP reporting pipe: "
                                        + ex.getMessage());
                            }

                            continue;
                        }

                        if (pollIndex > usapPoolEventFDIndex) {
                            Zygote.removeUsapTableEntry((int) messagePayload);
                        }

                        usapPoolFDRead = true;
                    }
                }

                if (usapPoolFDRead) {
                    int usapPoolCount = Zygote.getUsapPoolCount();

                    if (usapPoolCount < mUsapPoolSizeMin) {
                        // Immediate refill
                        mUsapPoolRefillAction = UsapPoolRefillAction.IMMEDIATE;
                    } else if (mUsapPoolSizeMax - usapPoolCount >= mUsapPoolRefillThreshold) {
                        // Delayed refill
                        mUsapPoolRefillTriggerTimestamp = System.currentTimeMillis();
                    }
                }
            }

            if (mUsapPoolRefillAction != UsapPoolRefillAction.NONE) {
                int[] sessionSocketRawFDs =
                        socketFDs.subList(1, socketFDs.size())
                                .stream()
                                .mapToInt(FileDescriptor::getInt$)
                                .toArray();

                final boolean isPriorityRefill =
                        mUsapPoolRefillAction == UsapPoolRefillAction.IMMEDIATE;

                final Runnable command =
                        fillUsapPool(sessionSocketRawFDs, isPriorityRefill);

                if (command != null) {
                    return command;
                } else if (isPriorityRefill) {
                    // Schedule a delayed refill to finish refilling the pool.
                    mUsapPoolRefillTriggerTimestamp = System.currentTimeMillis();
                }
            }
        }
    }
```

