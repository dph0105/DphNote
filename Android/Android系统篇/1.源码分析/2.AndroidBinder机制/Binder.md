Binder RPC调用的过程分析

### 例子

AIDL可以帮助我们创建Binder对象，通过bindService，我们可以在client进程调用server进程提供的方法。

```java
interface IFormatAidlInterface {
    String int2String(int number);
}
```

如上的AIDL文件，我们在bindService后的onServiceConnected回调中，通过 `IFormatAidlInterface.Stub.asInterface(service)`获取IFormatAidlInterface对象，并可以调用int2String方法让server进程进行int转String的操作。

```java
        public void onServiceConnected(ComponentName name, IBinder service) {
                iFormatAidlInterface= IFormatAidlInterface.Stub.asInterface(service);
                try {
                    String str = iFormatAidlInterface.int2String(1);
                    Log.d("Binder",str);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
        }
```

onServiceConnected中获取到的IBinder对象service实际上是一个BinderProxy对象，为什么呢？我们最后再讲，这里我们要记住，**server进程提供了Binder对象，传送到client进程后就转化为BinderProxy对象**

### 获取BinderProxy asInterface

通过 IFormatAidlInterface.Stub.asInterface我们将获取到的BinderProxy对象转化为一个IFormatAidlInterface对象，如何转化的呢？

```java
    public static com.ding.service.IFormatAidlInterface asInterface(android.os.IBinder obj)
    {
      if ((obj==null)) {
        return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin!=null)&&(iin instanceof com.ding.service.IFormatAidlInterface))) {
        return ((com.ding.service.IFormatAidlInterface)iin);
      }
      return new com.ding.service.IFormatAidlInterface.Stub.Proxy(obj);
    }
```

这里调用了 BinderProxy中的queryLocalInterface方法

```java
    public IInterface queryLocalInterface(String descriptor) {
        return null;
    }
```

BinderProxy中该方法直接返回null，因此在asInterface方法中会创建IFormatAidlInterface.Stub.Proxy对象，并传入BinderProxy对象作为mRemote成员变量。

```java
    private static class Proxy implements com.ding.service.IFormatAidlInterface
    {
      private android.os.IBinder mRemote;
      Proxy(android.os.IBinder remote)
      {
        mRemote = remote;
      }
      //......
    }
```

因此，asInterface中最终获取到的是一个Proxy对象。

### 使用BinderProxy开始远程调用

我们调用Proxy的int2String方法

```java
      @Override public java.lang.String int2String(int number) throws android.os.RemoteException
      {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        java.lang.String _result;
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          _data.writeInt(number);
          boolean _status = mRemote.transact(Stub.TRANSACTION_int2String, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            return getDefaultImpl().int2String(number);
          }
          _reply.readException();
          _result = _reply.readString();
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
        return _result;
      }
```

可以看到这里使用了Parcel来包裹数据，然后调用BinderProxy的transact方法进行RPC调用，这里获取两个Parcel对象，分别为 _data 用于包装参数以及 _reply用于包装返回的结果

#### 简单介绍Parcel

Parcel是Binder机制中发送的消息的容器，这里的消息指的是（数据或者对象），Parcel可以写入读取Byte、Int、Long、Float、Double、String这些基本数据类型

实际上，在Native层同样也有一个Parcel类，而Java层的Parce是Native Parcel的一个**“代理”**。Java Parcel中的成员变量mNativePtr即是Native Parcel类的对象的地址。

------

回到BinderProxy的transact方法中，这里会调用Native层的代码

```java
    public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        //不能发送大于800k的数据，默认是不检查的
        Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
		//......
        try {
            return transactNative(code, data, reply, flags);
        } finally {
			//......
        }
    }

```

### 进入Native层进程远程调用

Java层的BinderProxy方法调用transactNative方法进入Native层环节，这个Native方法定义在frameworks\base\core\jni\android_util_Binder.cpp中：

```c++
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
        jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException
{
    if (dataObj == NULL) {
        jniThrowNullPointerException(env, NULL);
        return JNI_FALSE;
    }
    //1.首先根据java对象获得本地对象
	//1.1根据包装参数的Java Parcel获取Native Parcel	
    Parcel* data = parcelForJavaObject(env, dataObj);
    if (data == NULL) {
        return JNI_FALSE;
    }
    //1.2根据包装返回结果的Java Parcel获取Native Parcel	
    Parcel* reply = parcelForJavaObject(env, replyObj);
    if (reply == NULL && replyObj != NULL) {
        return JNI_FALSE;
    }
	//1.3根据Java层BinderProxy获取Native BpBinder
    IBinder* target = getBPNativeData(env, obj)->mObject.get();
    if (target == NULL) {
        jniThrowException(env, "java/lang/IllegalStateException", "Binder has been finalized!");
        return JNI_FALSE;
    }
    //......
   	//2.调用Native BpBinder的transact函数
    status_t err = target->transact(code, *data, reply, flags);

	//......
    return JNI_FALSE;
}

```

#### 1、根据Java层对象获取Native对象

##### 1.1 - 1.2 获取Native Parcel

先看代码注释中的1.1 和 1.2 这里通过了Java 层用于传输数据和用于返回结果的两个Parcel对象获取到Native的这两个对象，获取的函数为parcelForJavaObject：

```c++
Parcel* parcelForJavaObject(JNIEnv* env, jobject obj)
{
    if (obj) {
        Parcel* p = (Parcel*)env->GetLongField(obj, gParcelOffsets.mNativePtr);
        if (p != NULL) {
            return p;
        }
        jniThrowException(env, "java/lang/IllegalStateException", "Parcel has been finalized!");
    }
    return NULL;
}
```

这里将JNI方法GetLongField中获得的地址转化为Native Parcel对象，gParcelOffsets.mNativePtr是在Android系统开机时Zygote进程启动时，会调用AndroidRuntime的start函数，其中会调用startReg函数进行一些Jni函数的参数配置，其中就包含Parcel的配置：

```c++
const char* const kParcelPathName = "android/os/Parcel";

int register_android_os_Parcel(JNIEnv* env)
{
    jclass clazz = FindClassOrDie(env, kParcelPathName);

    gParcelOffsets.clazz = MakeGlobalRefOrDie(env, clazz);
    gParcelOffsets.mNativePtr = GetFieldIDOrDie(env, clazz, "mNativePtr", "J");
    gParcelOffsets.obtain = GetStaticMethodIDOrDie(env, clazz, "obtain", "()Landroid/os/Parcel;");
    gParcelOffsets.recycle = GetMethodIDOrDie(env, clazz, "recycle", "()V");

    return RegisterMethodsOrDie(env, kParcelPathName, gParcelMethods, NELEM(gParcelMethods));
}
```

可以看到，gParcelOffsets.mNativePtr这里就是指向Java的Parcle类中的mNativePtr成员变量

##### 1.3 获取Native BpBinder

首先看getBPNativeData函数：

```c++
BinderProxyNativeData* getBPNativeData(JNIEnv* env, jobject obj) {
    return (BinderProxyNativeData *) env->GetLongField(obj, gBinderProxyOffsets.mNativeData);
}
```

与获取Native Parcel对象类似，这里也是通过JNI方法获取内存地址转化为BinderProxyNativeData对象，gBinderProxyOffsets.mNativeData同样也是在AndroidRuntime中的startReg方法中注册的：

```c++
int register_android_os_Binder(JNIEnv* env)
{
    if (int_register_android_os_Binder(env) < 0)
        return -1;
    if (int_register_android_os_BinderInternal(env) < 0)
        return -1;
    if (int_register_android_os_BinderProxy(env) < 0)//注册BinderProxy相关配置
        return -1;

    jclass clazz = FindClassOrDie(env, "android/util/Log");
    gLogOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    gLogOffsets.mLogE = GetStaticMethodIDOrDie(env, clazz, "e",
            "(Ljava/lang/String;Ljava/lang/String;Ljava/lang/Throwable;)I");

    clazz = FindClassOrDie(env, "android/os/ParcelFileDescriptor");
    gParcelFileDescriptorOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    gParcelFileDescriptorOffsets.mConstructor = GetMethodIDOrDie(env, clazz, "<init>",
                                                                 "(Ljava/io/FileDescriptor;)V");

    clazz = FindClassOrDie(env, "android/os/StrictMode");
    gStrictModeCallbackOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    gStrictModeCallbackOffsets.mCallback = GetStaticMethodIDOrDie(env, clazz,
            "onBinderStrictModePolicyChange", "(I)V");

    clazz = FindClassOrDie(env, "java/lang/Thread");
    gThreadDispatchOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    gThreadDispatchOffsets.mDispatchUncaughtException = GetMethodIDOrDie(env, clazz,
            "dispatchUncaughtException", "(Ljava/lang/Throwable;)V");
    gThreadDispatchOffsets.mCurrentThread = GetStaticMethodIDOrDie(env, clazz, "currentThread",
            "()Ljava/lang/Thread;");

    return 0;
}
const char* const kBinderProxyPathName = "android/os/BinderProxy";
static int int_register_android_os_BinderProxy(JNIEnv* env)
{
    gErrorOffsets.mError = MakeGlobalRefOrDie(env, FindClassOrDie(env, "java/lang/Error"));
    gErrorOffsets.mOutOfMemory =
        MakeGlobalRefOrDie(env, FindClassOrDie(env, "java/lang/OutOfMemoryError"));
    gErrorOffsets.mStackOverflow =
        MakeGlobalRefOrDie(env, FindClassOrDie(env, "java/lang/StackOverflowError"));

    jclass clazz = FindClassOrDie(env, kBinderProxyPathName);
    gBinderProxyOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    gBinderProxyOffsets.mGetInstance = GetStaticMethodIDOrDie(env, clazz, "getInstance",
            "(JJ)Landroid/os/BinderProxy;");
    gBinderProxyOffsets.mSendDeathNotice =
            GetStaticMethodIDOrDie(env, clazz, "sendDeathNotice",
                                   "(Landroid/os/IBinder$DeathRecipient;Landroid/os/IBinder;)V");
    gBinderProxyOffsets.mNativeData = GetFieldIDOrDie(env, clazz, "mNativeData", "J");

    clazz = FindClassOrDie(env, "java/lang/Class");
    gClassOffsets.mGetName = GetMethodIDOrDie(env, clazz, "getName", "()Ljava/lang/String;");

    return RegisterMethodsOrDie(
        env, kBinderProxyPathName,
        gBinderProxyMethods, NELEM(gBinderProxyMethods));
}
```

可以看到，gBinderProxyOffsets.mNativeData指向了Java层BinderProxy的mNativeData成员变量。

BinderProxyNativeData是一个结构体，包含了mObject指向一个Native的IBinder对象（这个IBinder对象是一个BpBinder对象），以及一个BpBinder对象死亡通知的列表。

```c++
struct BinderProxyNativeData {
    // The native IBinder proxied by this BinderProxy.
    sp<IBinder> mObject;

    sp<DeathRecipientList> mOrgue;  // Death recipients for mObject.
};
```

mObject是一个sp对象，sp是Android Native层中的类似强引用的对象，通过get方法获取到的BpBinder对象作为target。最后调用Native BpBinder的transact函数

#### 2、调用BpBinder的transact函数

```c++
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Binder存活的情况下，进行调用，否则直接返回
    if (mAlive) {
		//......
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;

        return status;
    }
    return DEAD_OBJECT;
}
```

BpBinder::transact调用IPCThreadState的transact，**IPCThreadState负责与Binder驱动进行具体的命令交互**

```c++
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err;

    flags |= TF_ACCEPT_FDS;
	//......
    //1.将传输的参数写入binder_transaction_data结构体中
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);

    if (err != NO_ERROR) {
        if (reply) reply->setError(err);
        return (mLastError = err);
    }
	//如果有TF_ONE_WAY标识，则是一个同步的进程间通信请求
    //2.不管如何下面的waitForResponse会向Binder驱动程序发送BC_TRANSACTION命令
    if ((flags & TF_ONE_WAY) == 0) {
		//.....
        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
		//......
    } else {
        err = waitForResponse(nullptr, nullptr);
    }

    return err;
}

```



##### 2.1 将数据存入缓冲区 writeTransactionData

```c++
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.ptr = 0;
    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        //若数据无错误，则填入结构体中
        tr.data_size = data.ipcDataSize();
        tr.data.ptr.buffer = data.ipcData();
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
        tr.data.ptr.offsets = data.ipcObjects();
    } else if (statusBuffer) {
        tr.flags |= TF_STATUS_CODE;
        *statusBuffer = err;
        tr.data_size = sizeof(status_t);
        tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer);
        tr.offsets_size = 0;
        tr.data.ptr.offsets = 0;
    } else {
        return (mLastError = err);
    }
	//将命令存入mOut缓冲区中
    mOut.writeInt32(cmd);
    //将binder_transaction_data结构体存入mOut缓冲区中
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}
```

writeTransactionData函数首先检查数据是否有误 ，无错的情况下，将命令以及根据参数组成的结构体存入mOut缓冲区中

IPCThreadState除了使用缓冲区mOut来保存即将要发送给Binder驱动程序的命令之外，还是用mIn缓冲区来保存从Binder驱动程序接收到的返回内容。

##### 2.2 向Binder驱动发送命令

```c++
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;
		
        // ......
    }

finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
    }

    return err;
}

```

waitForResponse在一个while循环中不断调用成员函数talkWithDriver来与Binder驱动程序进行交互，以便可以将在writeTransactionData函数中准备好的BC_TRANSACTION命令发送给Binder驱动程序进行处理，并等待Binder驱动程序将进程间通讯结果返回来

```c++
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    if (mProcess->mDriverFD < 0) {
        return -EBADF;
    }
	//定义结构体binder_write_read
    binder_write_read bwr;

    const bool needRead = mIn.dataPosition() >= mIn.dataSize();

    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
	
    //将缓冲区mOut的内容以及大小写入结构体bwr的相应变量中
    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    if (doReceive && needRead) {
         //若有返回，将缓冲区mIn的内容以及大小写入结构体bwr的相应变量中
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    //读写都无数据则直接返回
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
#if defined(__ANDROID__)
        //通过IO控制命令BINDER_WRITE_READ与Binder驱动进行交互
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
#else
        err = INVALID_OPERATION;
#endif
        if (mProcess->mDriverFD < 0) {
            err = -EBADF;
        }
        IF_LOG_COMMANDS() {
            alog << "Finished read/write, write size = " << mOut.dataSize() << endl;
        }
    } while (err == -EINTR);

    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < mOut.dataSize())
                LOG_ALWAYS_FATAL("Driver did not consume write buffer. "
                                 "err: %s consumed: %zu of %zu",
                                 statusToString(err).c_str(),
                                 (size_t)bwr.write_consumed,
                                 mOut.dataSize());
            else {
                mOut.setDataSize(0);
                processPostWriteDerefs();
            }
        }
        if (bwr.read_consumed > 0) {
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        } 
        return NO_ERROR;
    }

    return err;
}
```

talkWithDriver通过IO控制命令BINDER_WRITE_READ与Binder驱动进行交互

#### Binder驱动处理命令

*Android kernel中\drivers\android\binder.c* 定义了binder_ioctl函数处理命令，

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	//......

	switch (cmd) {
	case BINDER_WRITE_READ:
		ret = binder_ioctl_write_read(filp, cmd, arg, thread);
		if (ret)
			goto err;
		break;
	}
	ret = 0;
	//......
	return ret;
}

```

当接收到BINDER_WRITE_READ命令时，会调用binder_ioctl_write_read函数

```c
static int binder_ioctl_write_read(struct file *filp,
				unsigned int cmd, unsigned long arg,
				struct binder_thread *thread)
{
	int ret = 0;
	struct binder_proc *proc = filp->private_data;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;
	struct binder_write_read bwr;

	if (size != sizeof(struct binder_write_read)) {
		ret = -EINVAL;
		goto out;
	}
    //由于binder驱动运行在内核空间，copy_from_user将数据拷贝到内核空间
	if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}

	if (bwr.write_size > 0) {
        //1.调用binder_thread_write处理BC_TRANSACTION命令
		ret = binder_thread_write(proc, thread,
					  bwr.write_buffer,
					  bwr.write_size,
					  &bwr.write_consumed);
		trace_binder_write_done(ret);
        //若这次进程间通信不需要返回值，则直接返回，拷贝数据到用户空间
		if (ret < 0) {
			bwr.read_consumed = 0;
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
	if (bwr.read_size > 0) {
        //2.调用binder_thread_read读取Binder驱动的返回值
		ret = binder_thread_read(proc, thread, bwr.read_buffer,
					 bwr.read_size,
					 &bwr.read_consumed,
					 filp->f_flags & O_NONBLOCK);
		trace_binder_read_done(ret);
		binder_inner_proc_lock(proc);
		if (!binder_worklist_empty_ilocked(&proc->todo))
			binder_wakeup_proc_ilocked(proc);
		binder_inner_proc_unlock(proc);
		if (ret < 0) {
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
	if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}
out:
	return ret;
}
```



##### 1. binder_thread_write

```c

static int binder_thread_write(struct binder_proc *proc,
			struct binder_thread *thread,
			binder_uintptr_t binder_buffer, size_t size,
			binder_size_t *consumed)
{
	uint32_t cmd;
	struct binder_context *context = proc->context;
	void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
	void __user *ptr = buffer + *consumed;
	void __user *end = buffer + size;

	while (ptr < end && thread->return_error.cmd == BR_OK) {
        if (get_user(cmd, (uint32_t __user *)ptr))
			return -EFAULT;
		ptr += sizeof(uint32_t);
		switch (cmd) {
        //......
		case BC_TRANSACTION:
		case BC_REPLY: {
			struct binder_transaction_data tr;
			//读取用户空间中的binder_transaction_data数据
			if (copy_from_user(&tr, ptr, sizeof(tr)))
				return -EFAULT;
			ptr += sizeof(tr);
            //调用binder_transaction处理命令
			binder_transaction(proc, thread, &tr,
					   cmd == BC_REPLY, 0);
			break;
		}
	}
	return 0;
}

```



binder_transaction用于处理BC_TRANSACTION和BC_REPLY命令，参数reply用于判断是否是BC_REPLY命令。

```c
static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,
			       struct binder_transaction_data *tr, int reply,
			       binder_size_t extra_buffers_size)
{
	int ret;
	//......
	if (reply) {
		//......
	} else {
		if (tr->target.handle) {
			struct binder_ref *ref;

			binder_proc_lock(proc);
			ref = binder_get_ref_olocked(proc, tr->target.handle,
						     true);
			if (ref) {
				target_node = binder_get_node_refs_for_txn(
						ref->node, &target_proc,
						&return_error);
			} else {
				binder_user_error("%d:%d got transaction to invalid handle\n",
						  proc->pid, thread->pid);
				return_error = BR_FAILED_REPLY;
			}
			binder_proc_unlock(proc);
		} else {
			mutex_lock(&context->context_mgr_node_lock);
			target_node = context->binder_context_mgr_node;
			if (target_node)
				target_node = binder_get_node_refs_for_txn(
						target_node, &target_proc,
						&return_error);
			else
				return_error = BR_DEAD_REPLY;
			mutex_unlock(&context->context_mgr_node_lock);
			if (target_node && target_proc->pid == proc->pid) {
				binder_user_error("%d:%d got transaction to context manager from process owning it\n",
						  proc->pid, thread->pid);
				return_error = BR_FAILED_REPLY;
				return_error_param = -EINVAL;
				return_error_line = __LINE__;
				goto err_invalid_target_handle;
			}
		}
		if (!target_node) {
			/*
			 * return_error is set above
			 */
			return_error_param = -EINVAL;
			return_error_line = __LINE__;
			goto err_dead_binder;
		}
		e->to_node = target_node->debug_id;
		if (WARN_ON(proc == target_proc)) {
			return_error = BR_FAILED_REPLY;
			return_error_param = -EINVAL;
			return_error_line = __LINE__;
			goto err_invalid_target_handle;
		}
		if (security_binder_transaction(proc->tsk,
						target_proc->tsk) < 0) {
			return_error = BR_FAILED_REPLY;
			return_error_param = -EPERM;
			return_error_line = __LINE__;
			goto err_invalid_target_handle;
		}
		binder_inner_proc_lock(proc);

		w = list_first_entry_or_null(&thread->todo,
					     struct binder_work, entry);
		if (!(tr->flags & TF_ONE_WAY) && w &&
		    w->type == BINDER_WORK_TRANSACTION) {
			/*
			 * Do not allow new outgoing transaction from a
			 * thread that has a transaction at the head of
			 * its todo list. Only need to check the head
			 * because binder_select_thread_ilocked picks a
			 * thread from proc->waiting_threads to enqueue
			 * the transaction, and nothing is queued to the
			 * todo list while the thread is on waiting_threads.
			 */
			binder_user_error("%d:%d new transaction not allowed when there is a transaction on thread todo\n",
					  proc->pid, thread->pid);
			binder_inner_proc_unlock(proc);
			return_error = BR_FAILED_REPLY;
			return_error_param = -EPROTO;
			return_error_line = __LINE__;
			goto err_bad_todo_list;
		}

		if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) {
			struct binder_transaction *tmp;

			tmp = thread->transaction_stack;
			if (tmp->to_thread != thread) {
				spin_lock(&tmp->lock);
				binder_user_error("%d:%d got new transaction with bad transaction stack, transaction %d has target %d:%d\n",
					proc->pid, thread->pid, tmp->debug_id,
					tmp->to_proc ? tmp->to_proc->pid : 0,
					tmp->to_thread ?
					tmp->to_thread->pid : 0);
				spin_unlock(&tmp->lock);
				binder_inner_proc_unlock(proc);
				return_error = BR_FAILED_REPLY;
				return_error_param = -EPROTO;
				return_error_line = __LINE__;
				goto err_bad_call_stack;
			}
			while (tmp) {
				struct binder_thread *from;

				spin_lock(&tmp->lock);
				from = tmp->from;
				if (from && from->proc == target_proc) {
					atomic_inc(&from->tmp_ref);
					target_thread = from;
					spin_unlock(&tmp->lock);
					break;
				}
				spin_unlock(&tmp->lock);
				tmp = tmp->from_parent;
			}
		}
		binder_inner_proc_unlock(proc);
	}

	//......
```



