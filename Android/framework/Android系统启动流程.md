### Android系统启动流程

##### 1.启动电源以及系统启动

​	当电源按下时，应道芯片代码从ROM中执行，加载引导程序BootLoader到RAM中，然后执行。

##### 2.引导程序BootLoader

​	引导程序BootLoader是在Android操作系统开始运行前的一个小程序，它的主要作用是把系统OS拉起来并运行

##### 3. Linux内核启动

​	当内核启动时，设置缓存、被保护存储器、计划列表、加载驱动。当内核完成系统设置时，首先在系统中寻找init.rc文件，并启动init进程。

##### 4. Init进程

​	Init进程会运行init.rc脚本（位于system\core\rootdir），然后执行app_main.cpp中的main()函数(位于frameworks\base\cmds\app_process)。

```c++
//app_main.cpp
int main(int argc, char* const argv[])
{
    //......
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
	//......处理参数
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



在app_main.cpp的main函数中，会构造一个`AppRuntime`对象，在函数的最后，通过`start()`

函数调用`ZygoteInit.java`的main函数

```
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    /* 首先启动虚拟机 */
    JniInvocation jni_invocation;
    //通过JniInvocation初始化一些本地方法函数
    jni_invocation.Init(NULL);
    JNIEnv* env;
	//启动虚拟机
    if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) {
        return;
    }
    //虚拟机创建成功的回调函数
    onVmCreated(env);

    /*
     * Register android functions.
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

    /*
     * We want to call main() with a String array with arguments in it.
     * At present we have two arguments, the class name and an option string.
     * Create an array to hold them.
     */
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);

    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }

    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */

	//slashClassName 就是传入的com.android.internal.os.ZygoteInit 将.替换为/ 
	//com/android/internal/os/ZygoteInit
    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    jclass startClass = env->FindClass(slashClassName);//通过JNI找到ZygoteInit这个类
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        //找到类后，找类中的main()方法
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
        	//找到方法后，调用该方法
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



