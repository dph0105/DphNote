# 前言

SurfaceFlinger是Android 图形架构中承上启下的一个重要组件，它接收来自多个源的缓冲数据（例如 WindowManager），并将他们发送到显示设备。

了解这一过程有助于我们对Android整个绘制过程有所帮助

# SurfaceFlinger启动

当Android系统启动时，首先启动 init 进程，init进程运行 init.rc 脚本文件，surfaceflinger进程从这里被启动。

\frameworks\native\services\surfaceflinger\surfaceflinger.rc

```c++
service surfaceflinger /system/bin/surfaceflinger
    class core animation
    user system
    group graphics drmrpc readproc
    capabilities SYS_NICE
    onrestart restart zygote
    task_profiles HighPerformance
    socket pdx/system/vr/display/client     stream 0666 system graphics u:object_r:pdx_display_client_endpoint_socket:s0
    socket pdx/system/vr/display/manager    stream 0666 system graphics u:object_r:pdx_display_manager_endpoint_socket:s0
    socket pdx/system/vr/display/vsync      stream 0666 system graphics u:object_r:pdx_display_vsync_endpoint_socket:s0
```

surfaceflinger.rc定义了surfaceflinger服务，服务对应的执行文件在/system/bin/surfaceflinger，其源码在 \frameworks\native\services\surfaceflinger\main_surfaceflinger.cpp，程序入口是 main_surfaceflinger.cpp 的main()方法:

```c++
int main(int, char**) {
    signal(SIGPIPE, SIG_IGN);
	//设置HIDL支持的最大线程数为1
    hardware::configureRpcThreadpool(1 /* maxThreads */,false /* callerWillJoin */);
	//启动图形分配器服务
    startGraphicsAllocatorService();

    // 当 SF 在自己的进程中启动时，将 binder 线程数限制为 4。
    ProcessState::self()->setThreadPoolMaxThreadCount(4);
    sp<ProcessState> ps(ProcessState::self());
    ps->startThreadPool();

    // 创建surfaceflinger实例,作为一个强引用指针
    sp<SurfaceFlinger> flinger = surfaceflinger::createSurfaceFlinger();

    setpriority(PRIO_PROCESS, 0, PRIORITY_URGENT_DISPLAY);
	//设置调度策略，SP_FOREGROUND优先级
    set_sched_policy(0, SP_FOREGROUND);

    //设置surfaceflinger的cpu策略，使用后台的，比较小核的cpu
    if (cpusets_enabled()) set_cpuset_policy(0, SP_SYSTEM);

    //初始化
    flinger->init();

    // 发布 SurfaceFlinger服务到 ServiceManager中
    sp<IServiceManager> sm(defaultServiceManager());
    sm->addService(String16(SurfaceFlinger::getServiceName()), flinger, false,
                   IServiceManager::DUMP_FLAG_PRIORITY_CRITICAL | IServiceManager::DUMP_FLAG_PROTO);
	
    //启动DisplayService
    startDisplayService(); // dependency on SF getting registered above

    if (SurfaceFlinger::setSchedFifo(true) != NO_ERROR) {
        ALOGW("Couldn't set to SCHED_FIFO: %s", strerror(errno));
    }

    // 在当前线程中运行 surfaceflinger，接收消息
    flinger->run();

    return 0;
}
```

## 1. 创建surfaceflinger实例

\frameworks\native\services\surfaceflinger\SurfacerFlingerFactory.cpp

```c++
sp<SurfaceFlinger> createSurfaceFlinger() {
    static DefaultFactory factory;
    return new SurfaceFlinger(factory);
}
```

DefaultFactory定义于 \frameworks\native\services\surfaceflinger\SurfaceFlingerDefaultFactory.cpp。它的作用是用于创建SurfaceFlinger中使用到的一些对象。

通过new的方式创建了一个SurfaceFlinger对象，传入了DefalutFactory对象：

```c++

SurfaceFlinger::SurfaceFlinger(Factory& factory, SkipInitializationTag)
      : mFactory(factory),
        mInterceptor(mFactory.createSurfaceInterceptor(this)),
        mTimeStats(std::make_shared<impl::TimeStats>()),
        mFrameTracer(std::make_unique<FrameTracer>()),
        mEventQueue(mFactory.createMessageQueue()),
        mCompositionEngine(mFactory.createCompositionEngine()),
        mInternalDisplayDensity(getDensityFromProperty("ro.sf.lcd_density", true)),
        mEmulatedDisplayDensity(getDensityFromProperty("qemu.sf.lcd_density", false)) {}

SurfaceFlinger::SurfaceFlinger(Factory& factory) : SurfaceFlinger(factory, SkipInitialization) {
	//......
}
```

可以看到这里通过 SurfaceFlingerDefaultFactory 创建了 mEventQueue 和 mCompositionEngine

```c++
std::unique_ptr<MessageQueue> DefaultFactory::createMessageQueue() {
    return std::make_unique<android::impl::MessageQueue>();
}

std::unique_ptr<compositionengine::CompositionEngine> DefaultFactory::createCompositionEngine() {
    return compositionengine::impl::createCompositionEngine();
}
```

MessageQueue用于消息循环

CompositionEngine创建：

```c++
std::unique_ptr<compositionengine::CompositionEngine> createCompositionEngine() {
    return std::make_unique<CompositionEngine>();
}
```

由于创建的SurfaceFlinger实例是一个sp，即强引用指针，在指针对象初始化中，会调用 onFirstRef() 方法初始化MessageQueue的Looper和Handler

```c++
//\frameworks\native\services\surfaceflinger\SurfaceFlinger.cpp
void SurfaceFlinger::onFirstRef() {
    mEventQueue->init(this);
}
//\frameworks\native\services\surfaceflinger\Scheduler\MessageQueue.cpp
void MessageQueue::init(const sp<SurfaceFlinger>& flinger) {
    mFlinger = flinger;
    mLooper = new Looper(true);
    mHandler = new Handler(*this);
}
```

## 2. SurfaceFlinger初始化

上一步创建了SurfaceFlinger对象之后，main方法中会调用SurfaceFlinger的初始化方法 init()

```c++
void SurfaceFlinger::init() {
    Mutex::Autolock _l(mStateLock);
	//为CompositionEngine设置一个RenderEngine
    mCompositionEngine->setRenderEngine(renderengine::RenderEngine::create(
            renderengine::RenderEngineCreationArgs::Builder()
                .setPixelFormat(static_cast<int32_t>(defaultCompositionPixelFormat))
                .setImageCacheSize(maxFrameBufferAcquiredBuffers)
                .setUseColorManagerment(useColorManagement)
                .setEnableProtectedContext(enable_protected_contents(false))
                .setPrecacheToneMapperShaderOnly(false)
                .setSupportsBackgroundBlur(mSupportsBlur)
                .setContextPriority(useContextPriority
                        ? renderengine::RenderEngine::ContextPriority::HIGH
                        : renderengine::RenderEngine::ContextPriority::MEDIUM)
                .build()));
    mCompositionEngine->setTimeStats(mTimeStats);

    // 初始化一个硬件HWComposer对象，然后通过CompositionEngine指针对象保存HWComposer指针对象地址引用
    mCompositionEngine->setHwComposer(getFactory().createHWComposer(getBE().mHwcServiceName));
    // 设置 HWComposer的 Configuration
    mCompositionEngine->getHwComposer().setConfiguration(this, getBE().mComposerSequenceId);
    // 处理任何初始热插拔和由此产生的显示更改。
    processDisplayHotplugEventsLocked();
    //获取默认显示设备
    const auto display = getDefaultDisplayDeviceLocked();
	//虚拟设备
    if (useVrFlinger) {
        auto vrFlingerRequestDisplayCallback = [this](bool requestDisplay) {
            static_cast<void>(schedule([=] {
                ALOGI("VR request display mode: requestDisplay=%d", requestDisplay);
                mVrFlingerRequestsDisplay = requestDisplay;
                signalTransaction();
            }));
        };
        mVrFlinger = dvr::VrFlinger::Create(getHwComposer().getComposer(),
                                            getHwComposer()
                                                    .fromPhysicalDisplayId(*display->getId())
                                                    .value_or(0),
                                            vrFlingerRequestDisplayCallback);
        if (!mVrFlinger) {
            ALOGE("Failed to start vrflinger");
        }
    }

    // 初始化图形状态
    mDrawingState = mCurrentState;

    // 初始化显示设备
    initializeDisplays();

    char primeShaderCache[PROPERTY_VALUE_MAX];
    property_get("service.sf.prime_shader_cache", primeShaderCache, "1");
    if (atoi(primeShaderCache)) {
        getRenderEngine().primeCache();
    }

    const bool presentFenceReliable =
            !getHwComposer().hasCapability(hal::Capability::PRESENT_FENCE_IS_NOT_RELIABLE);
    //创建一个StartPropertySetThread线程并运行
    mStartPropertySetThread = getFactory().createStartPropertySetThread(presentFenceReliable);
    if (mStartPropertySetThread->Start() != NO_ERROR) {
        ALOGE("Run StartPropertySetThread failed!");
    }
}

```



### 2.1 创建HWComposer

上一小节讲到，在创建SurfaceFlinger对象时，会传入一个factory，此时这个factory就发挥了作用，创建HWComposer对象：

```c++
std::unique_ptr<HWComposer> DefaultFactory::createHWComposer(const std::string& serviceName) {
    return std::make_unique<android::impl::HWComposer>(serviceName);
}
```

serviceName为 “default” 





















































































































