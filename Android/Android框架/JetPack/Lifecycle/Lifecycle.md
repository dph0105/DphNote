1. Lifecycle-Aware简介

Android Jetpack提供了Lifecycle-Aware组件帮助我们响应其他组件（Activity或Fragment）的生命周期变化。例如Activity，即使不适用Lifecycle-Aware，我们也可以在它的生命周期方法中执行需要的任务，但是这样导致了代码的条理性很差而且会扩散错误。

而通过Lifecycle-Aware组件，我们可以将依赖于生命周期的逻辑代码分离出去，我们来看一下具体的做法：


```java
    class MyObserver : LifecycleObserver {

        @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
        fun connectListener() {
            ...
        }

        @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
        fun disconnectListener() {
            ...
        }
    }
    
    myLifecycleOwner.getLifecycle().addObserver(MyObserver())
```

## 2. Lifecycle-Aware三大角色

通过上面这个小例子，我们可以发现Lifecycle-Aware组件有三个重要的类`LifecycleObserver`、`LifecycleOwner`以及`Lifecycle`。

### 2.1 LifecycleOwner

LifecycleOwner是一个接口，实现该接口，就表示类具有Lifecycle，**即这个类是有生命周期的**：

```java
public interface LifecycleOwner {
    @NonNull
    Lifecycle getLifecycle();
}
```

它只有一个方法，即getLifecycle()，像ComponentActivity、Fragment都实现了该接口，所以他们需要提供一个Lifecycle对象，我们可以先看ComponentActivity中的实现：


```java
public class ComponentActivity extends Activity implements
        LifecycleOwner,
        KeyEventDispatcher.Component {

    private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
    ...
}
```
### 2.2 Lifecycle

`Lifecycle`是一个抽象类，所以在`ComponentActivity`中创建的是它的一个子类`LifecycleRegistry`，并可以通过实现的LifecycleOwner方法——getLifecycle()获得它。


```java
public class LifecycleRegistry extends Lifecycle {
    //LifecycleRegistry不一定只有一个观察者，使用一个Map来缓存这些观察者
    private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap = new FastSafeIterableMap<>();
    
    //表示当前的生命周期状态
    private State mState;
  
    //用弱引用持有mLifecycleOwner，即Activity、Fragment等
    private final WeakReference<LifecycleOwner> mLifecycleOwner;

    private int mAddingObserverCounter = 0;

    private boolean mHandlingEvent = false;
    private boolean mNewEventOccurred = false;

    private ArrayList<State> mParentStates = new ArrayList<>();
    
    public LifecycleRegistry(@NonNull LifecycleOwner provider) {
        mLifecycleOwner = new WeakReference<>(provider);
        mState = INITIALIZED;//构造时，设置状态为初始状态INITIALIZED
    }
    
    ...
```
`LifecycleRegistry`的构造方法接受一个LifecycleOwner，使用弱引用包装它防止内存泄漏，`LifecycleRegistry`中使用mState变量记录生命周期组件的状态，初始化状态为**INITIALIZED**，它的类型是State，是一个枚举类：

```java
public enum State {
    DESTROYED,

    INITIALIZED,
    
    CREATED,

    STARTED,

    RESUMED;

    public boolean isAtLeast(@NonNull State state) {
        return compareTo(state) >= 0;
    }
}
```
这里我们看一下State的状态变化：


![image](https://developer.android.google.cn/images/topic/libraries/architecture/lifecycle-states.svg)



### 2.3 LifecycleObserver

**文章一开始的小例子**就是一个自定义的LifecycleObserver，我们需要为`LifecycleRegistry`添加一个`LifecycleObserver`去监听生命周期。`LifecycleObserver`是一个不带任何方法的接口，它用于标记一个类是LifecycleObserver（标记这个类为生命周期的观察者），它的实现需要依赖于用`OnLifecycleEvent`注解的方法。

```java
public interface LifecycleObserver {

}
```

## 3. 生命周期的通知

粗略的介绍完三个类之后，我们是否会想到，为什么LifecycleObserver可以接收到生命周期事件呢？我们应该能想到Lifecycle必须在生命周期事件的回调中，即时通知LifecycleObserver。

**首先我们必须认识到，生命周期事件的传递流程是从 LifecycleOwner（Activity、Fragment） ➡ Lifecycle（LifecycleRegistry） ➡  LifecycleObserver**。接下来我们慢慢分析

### 3.1 Lifecycle添加观察者LifecycleObserver

```java
public abstract class Lifecycle {
    @MainThread
    public abstract void addObserver(@NonNull LifecycleObserver observer);

    @MainThread
    public abstract void removeObserver(@NonNull LifecycleObserver observer);
 
    ...
}
```

Lifecycle中添加一个观察者依靠`addObserver`,LifecycleRegistry实现了该方法：

```java
    @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
         //1.创建ObserverWithState，并且设置状态为DESTROYED或者INITIALIZED
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        //2.以LifecycleObserver为key，ObserverWithState为value传入map中记录
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);
        //如果已经添加过，则直接返回
        if (previous != null) {
            return;
        }
        //mLifecycleOwner是WeakReference，持有Activity
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) { 
            return;
        }
		//判断是否代码重入      这个表示正在添加Observer         这个表示正在分发事件
        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        //3.计算此时的目标State
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        //4.这个循环的目的是将我们新建的ObserverWithState从初始State（DESTROYED或者INITIALIZED）
        //更新到目前的State
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            //在这个循环的过程中，或许LifecycleOwner（即Activity或Fragment）又改变了状态
            //所以重新计算
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            //
            sync();
        }
        mAddingObserverCounter--;
    }
```

addObserver(@NonNull LifecycleObserver observer)需要传入一个LifecycleObserver对象，而在方法中则首先通过LifecycleObserver得到一个**ObserverWithState**对象，并以传入的LifecycleObserver为key，以这个ObserverWithState为value，加入到mObserverMap中缓存起来。

#### 3.1.1 创建ObserverWithState

ObserverWithState顾名思义，是携带State的Observer，它是LifeRegistry的静态内部类：


```java
    static class ObserverWithState {
        State mState;
        LifecycleEventObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
            mState = initialState;
        }
		//ObserverWithState通过该方法分发事件
        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
            mState = min(mState, newState);
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }
```
`ObserverWithState`只有一个方法`dispatchEvent(LifecycleOwner owner, Event event)`，它用于分发生命周期事件。**Lifecycle通过该函数将生命周期事件传递到LifecycleObserver中。**

ObserverWithState只有两个成员变量，一个是State表示生命周期状态。另一个是LifecycleEventObserver，在ObserverWithState的构造方法中，通过`Lifecycling.lifecycleEventObserver(LifecycleObserver)`会将传入的LifecycleObserver转化为一个LifecycleEventObserver。

##### 3.1.1.1 创建LifecycleEventObserver

这里我们看到它的属性，一个State，一个LifecycleEventObserver，在`ObserverWithState`的构造方法中，通过`Lifecycling.lifecycleEventObserver(Object)`将传入的`LifecycleObserver`转化为一个`LifecycleEventObserver`。

```java
public interface LifecycleEventObserver extends LifecycleObserver {
    void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event);
}
```
LifecycleEventObserver是一个接口，继承LifecycleObserver。ObserverWithState的`dispatchEvent(LifecycleOwner owner, Event event)`就是通过它的`onStateChanged(LifecycleOwner source,Lifecycle.Event event)`将事件传递给我们。

**真正接收生命周期事件的是LifecycleEventObserver**

在文章开篇的例子中，是通过注解的方式得到的LifecycleObserver。除了这种方式之外，我们还可以直接实现LifecycleEventObserver，如下：

```kotlin
lifecycle.addObserver(object : LifecycleEventObserver{
    override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
        when(event){
            Lifecycle.Event.ON_CREATE -> Log.d("Lifecycle","on_create")
            Lifecycle.Event.ON_RESUME -> Log.d("Lifecycle","on_resume")
            else -> Log.d("Lifecycle","other")
        }
    }
})
```
 `onStateChanged(source:LifecycleOwner,event:Lifecycle.Event)`方法会在生命周期改变时调用，event为改变的生命周期事件。

通过ObserverWithState可以知道，最终LifecycleObserver会转化成LifecycleEventObserver，看它的转化方法`Lifecycling.lifecycleEventObserver(observer)`

```java
    @NonNull
    static LifecycleEventObserver lifecycleEventObserver(Object object) {
        //判断是否是LifecycleEventObserver或者LifecycleEventObserver
        boolean isLifecycleEventObserver = object instanceof LifecycleEventObserver;
        boolean isFullLifecycleObserver = object instanceof FullLifecycleObserver;
        if (isLifecycleEventObserver && isFullLifecycleObserver) {
            return new FullLifecycleObserverAdapter((FullLifecycleObserver) object,
                    (LifecycleEventObserver) object);
        }
        if (isFullLifecycleObserver) {
            return new FullLifecycleObserverAdapter((FullLifecycleObserver) object, null);
        }
        if (isLifecycleEventObserver) {
            return (LifecycleEventObserver) object;
        }
		//如果不是则要开始转化
        final Class<?> klass = object.getClass();
        int type = getObserverConstructorType(klass);
        if (type == GENERATED_CALLBACK) {
            List<Constructor<? extends GeneratedAdapter>> constructors =
                    sClassToAdapters.get(klass);
            if (constructors.size() == 1) {
                GeneratedAdapter generatedAdapter = createGeneratedAdapter(
                        constructors.get(0), object);
                return new SingleGeneratedAdapterObserver(generatedAdapter);
            }
            GeneratedAdapter[] adapters = new GeneratedAdapter[constructors.size()];
            for (int i = 0; i < constructors.size(); i++) {
                adapters[i] = createGeneratedAdapter(constructors.get(i), object);
            }
            return new CompositeGeneratedAdaptersObserver(adapters);
        }
        return new ReflectiveGenericLifecycleObserver(object);
    }
```

我们看到该方法的目的是得到一个LifecycleEventObserver对象，所以会先判断传入的Observer是否已经是LifecycleEventObserver类了。FullLifecycleObserver是一个不使用的类了，它内部包含全部的生命周期事件方法，这里就不细讲了。

**看一下创建LifecycleEventObserver的步骤：**

###### 1. 分析LifecycleObserver的类型

当传入的Observer不属于LifecycleEventObserver或者FullLifecycleObserver时，首先执行了`getObserverConstructorType(klass)`

```java
    private static final int REFLECTIVE_CALLBACK = 1;
    private static final int GENERATED_CALLBACK = 2;

    private static Map<Class<?>, Integer> sCallbackCache = new HashMap<>();

    private static int getObserverConstructorType(Class<?> klass) {
        Integer callbackCache = sCallbackCache.get(klass);
        if (callbackCache != null) {
            return callbackCache;
        }
        int type = resolveObserverCallbackType(klass);
        sCallbackCache.put(klass, type);
        return type;
    }
```

`getObserverConstructorType(klass)`返回一个int值，名为type，用于表示Observer的Class的类型，这里将Class类型分为两类`REFLECTIVE_CALLBACK`表示需要反射生成的类，`GENERATED_CALLBACK`表示会编译时会生成一个继承`GeneretedAdapter`的Observer。

得到type类型后，通过一个Map`sCallbackCache`将解析后的结果缓存起来，防止多次解析相同的类。像我们的例子中，返回的type就是REFLECTIVE_CALLBACK。

我们进入`resolveObserverCallbackType`方法中看看是如何解析的：


```java
    private static int resolveObserverCallbackType(Class<?> klass) {
        //1.如果没有类名，即匿名类，则直接返回REFLECTIVE_CALLBACK
        if (klass.getCanonicalName() == null) {
            return REFLECTIVE_CALLBACK;
        }
        //2.得到编译器构建的继承GeneratedAdapter的类的构造器，
        //若找到这个类，将构造器缓存进sClassToAdapters这个Map中，返回GENERATED_CALLBACK
        //文中的情况，这里返回null
        Constructor<? extends GeneratedAdapter> constructor = generatedConstructor(klass);
        if (constructor != null) {
            sClassToAdapters.put(klass, Collections
                    .<Constructor<? extends GeneratedAdapter>>singletonList(constructor));
            return GENERATED_CALLBACK;
        }
        //3.判断我们传入的Observer类中，是否有OnLifecycleEvent的注解方法
        //如果有注解方法，直接返回REFLECTIVE_CALLBACK
        boolean hasLifecycleMethods = ClassesInfoCache.sInstance.hasLifecycleMethods(klass);
        if (hasLifecycleMethods) {
            return REFLECTIVE_CALLBACK;
        }
        //4.递归调用getObserverConstructorType方法，判断Observer的父类
        Class<?> superclass = klass.getSuperclass();
        List<Constructor<? extends GeneratedAdapter>> adapterConstructors = null;
        if (isLifecycleParent(superclass)) {
            if (getObserverConstructorType(superclass) == REFLECTIVE_CALLBACK) {
                return REFLECTIVE_CALLBACK;
            }
            adapterConstructors = new ArrayList<>(sClassToAdapters.get(superclass));
        }
        //5.遍历klass实现的接口
        for (Class<?> intrface : klass.getInterfaces()) {
            if (!isLifecycleParent(intrface)) {
                continue;
            }
            if (getObserverConstructorType(intrface) == REFLECTIVE_CALLBACK) {
                return REFLECTIVE_CALLBACK;
            }
            if (adapterConstructors == null) {
                adapterConstructors = new ArrayList<>();
            }
            adapterConstructors.addAll(sClassToAdapters.get(intrface));
        }
        if (adapterConstructors != null) {
            sClassToAdapters.put(klass, adapterConstructors);
            return GENERATED_CALLBACK;
        }
      
        return REFLECTIVE_CALLBACK;
    }

```

该方法的过程有些长，但是我们可以看到主要是依靠generatedConstructor(klass)方法获得一个`GeneraterAddapter`的子类的构造方法，去判断我们传入的Observer是否属于**GENERATED_CALLBACK**，我们看一下该方法：


```java
    @Nullable
    private static Constructor<? extends GeneratedAdapter> generatedConstructor(Class<?> klass) {
        try {
           //得到包
            Package aPackage = klass.getPackage();
            //得到全类名，如我们类名为MyObserver，则name为com.xx.xx.MyObserver
            String name = klass.getCanonicalName();
            //包名
            final String fullPackage = aPackage != null ? aPackage.getName() : "";
            //得到Adapter的名称，即MyObserver_LifecycleAdapter
            final String adapterName = getAdapterName(fullPackage.isEmpty() ? name :
                    name.substring(fullPackage.length() + 1));
            //获得MyObserver_LifecycleAdapter的Class对象
            @SuppressWarnings("unchecked") final Class<? extends GeneratedAdapter> aClass =
                    (Class<? extends GeneratedAdapter>) Class.forName(
                            fullPackage.isEmpty() ? adapterName : fullPackage + "." + adapterName);
            //获取MyObserver_LifecycleAdapter的构造方法
            Constructor<? extends GeneratedAdapter> constructor =
                    aClass.getDeclaredConstructor(klass);
            if (!constructor.isAccessible()) {
                //设置访问权限
                constructor.setAccessible(true);
            }
            return constructor;
        } catch (ClassNotFoundException e) {
            //找不到类返回null
            return null;
        } catch (NoSuchMethodException e) {
            // this should not happen
            throw new RuntimeException(e);
        }
    }

```
我们generatedConstructor会根据传入的类型，得到编译时，自动生成的带有我们自己创建的Observer的类名+_LifecycleAdapter后缀的一个GeneraterAdatper的子类的构造方法。当然，如果没有这个类，则返回null，例如文中的情况。

由于我几次尝试下，都没有生成这样的类，所以GENERATED_CALLBACK类型的情况只分析到这里。像我这种情况（两种用法：匿名内部类和自定义类实现LifecycleObserver）返回的type为`REFLECTIVE_CALLBACK`

###### 2. 根据类型创建ReflectiveGenericLifecycleObserver

即ObserverWithState中的LifecycleEventObserver对象实际为一个`ReflectiveGenericLifecycleObserver`。我们来看这个类的代码：

```java
class ReflectiveGenericLifecycleObserver implements LifecycleEventObserver {
    private final Object mWrapped;
    private final CallbackInfo mInfo;

    ReflectiveGenericLifecycleObserver(Object wrapped) {
        mWrapped = wrapped;
        mInfo = ClassesInfoCache.sInstance.getInfo(mWrapped.getClass());
    }

    @Override
    public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Event event) {
        mInfo.invokeCallbacks(source, event, mWrapped);
    }
}
```

我们看到`ReflectiveGenericLifecycleObserver`实现了`LifecycleEventObserver`，在它的构造器中，创建了一个`CallbackInfo`对象，并且`onStateChanged()`方法的实现是依靠`CallbackInfo.invokeCallbacks()`方法实现的。

**首先在`ReflectiveGenericLifecycleObserver`的构造方法中传入的`wrapped`是我们创建的LifycycleObserver**，我们进入ClassesInfoCache.sInstance是ClassesInfoCache的单例，接下来我们继续进入getInfo()方法中：


```java
    CallbackInfo getInfo(Class<?> klass) {
        CallbackInfo existing = mCallbackMap.get(klass);
        if (existing != null) {
            return existing;
        }
        existing = createInfo(klass, null);
        return existing;
    }
```
首先从缓存Map mCallbackMap中取存在的CallbackInfo，不存在则创建一个新的，我们直接进入createInfo方法中：


```java
   private CallbackInfo createInfo(Class<?> klass, @Nullable Method[] declaredMethods) {
        //遍历父类，调用getInfo方法，记录定义的方法到handlerToEvent
        Class<?> superclass = klass.getSuperclass();
        Map<MethodReference, Lifecycle.Event> handlerToEvent = new HashMap<>();
        if (superclass != null) {
            CallbackInfo superInfo = getInfo(superclass);
            if (superInfo != null) {
                handlerToEvent.putAll(superInfo.mHandlerToEvent);
            }
        }
        //2.遍历接口，调用getInfo方法，记录定义的方法到handlerToEvent
        Class<?>[] interfaces = klass.getInterfaces();
        for (Class<?> intrfc : interfaces) {
            for (Map.Entry<MethodReference, Lifecycle.Event> entry : getInfo(
                    intrfc).mHandlerToEvent.entrySet()) {
                verifyAndPutHandler(handlerToEvent, entry.getKey(), entry.getValue(), klass);
            }
        }
        //3.循环类中的方法，作用是判断方法的样式，即无参，一个参数，两个参数
        Method[] methods = declaredMethods != null ? declaredMethods : getDeclaredMethods(klass);
        boolean hasLifecycleMethods = false;
        for (Method method : methods) {
            //3.1若方法没有OnLifecycleEvent注解，直接跳过
            OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
            if (annotation == null) {
                continue;
            }
            hasLifecycleMethods = true;
            Class<?>[] params = method.getParameterTypes();
            //3.2首先定义方法类型为CALL_TYPE_NO_ARG，即没有参数
            int callType = CALL_TYPE_NO_ARG;
            //3.3若方法有参数，则方法类型设置为CALL_TYPE_PROVIDER
            //并且第一个参数必须为LifecycleOwner
            if (params.length > 0) {
                callType = CALL_TYPE_PROVIDER;
                if (!params[0].isAssignableFrom(LifecycleOwner.class)) {
                    throw new IllegalArgumentException(
                            "invalid parameter type. Must be one and instanceof LifecycleOwner");
                }
            }
            Lifecycle.Event event = annotation.value();
            //3.4若方法有两个参数，则方法类型为CALL_TYPE_PROVIDER_WITH_EVENT
            //并且第二个参数必须为Lifecycle.Event.ON_ANY
            //且OnLifecycleEvent注解必须为Lifecycle.Event.ON_ANY
            if (params.length > 1) {
                callType = CALL_TYPE_PROVIDER_WITH_EVENT;
                if (!params[1].isAssignableFrom(Lifecycle.Event.class)) {
                    throw new IllegalArgumentException(
                            "invalid parameter type. second arg must be an event");
                }
                if (event != Lifecycle.Event.ON_ANY) {
                    throw new IllegalArgumentException(
                            "Second arg is supported only for ON_ANY value");
                }
            }
            //若参数大于2个，则抛异常
            if (params.length > 2) {
                throw new IllegalArgumentException("cannot have more than 2 params");
            }
            //3.5将方法类型callType和方法method封装为一个MethodReference
            MethodReference methodReference = new MethodReference(callType, method);
            //验证是否已经添加，验证通过会以MethodReference为key，Method为value加入到handlerToEvent中
            verifyAndPutHandler(handlerToEvent, methodReference, event, klass);
        }
        //创建CallbackInfo，传入我们解析的方法Map
        CallbackInfo info = new CallbackInfo(handlerToEvent);
        //缓存
        mCallbackMap.put(klass, info);
        mHasLifecycleMethods.put(klass, hasLifecycleMethods);
        return info;
    }

```

createInfo()方法虽然有些长，但是流程却很清楚，CallbackInfo其实就是包装着我们定义的生命周期事件回调的方法。并且，我们根据代码，可以自由的定义我们Observer的写法：


```java
    //没有参数
    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    fun onCreate() {
        Log.d("Lifecycle","ON_CREATE")
    }

    //一个参数
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    fun onCreate(owner: LifecycleOwner) {
         Log.d("Lifecycle","ON_RESUME")
    }
    
    //两个参数
    @OnLifecycleEvent(Lifecycle.Event.ON_ANY)
    fun any(owner: LifecycleOwner,event: Lifecycle.Event) {
        when(event){
            Lifecycle.Event.ON_CREATE -> Log.d("Lifecycle","ON_CREATE")
            else -> Log.d("Lifecycle","else")
        }
    }
```

我们看一下CallbackInfo这个类：


```java
    static class CallbackInfo {
        final Map<Lifecycle.Event, List<MethodReference>> mEventToHandlers;
        final Map<MethodReference, Lifecycle.Event> mHandlerToEvent;

        CallbackInfo(Map<MethodReference, Lifecycle.Event> handlerToEvent) {
            mHandlerToEvent = handlerToEvent;
            mEventToHandlers = new HashMap<>();
            for (Map.Entry<MethodReference, Lifecycle.Event> entry : handlerToEvent.entrySet()) {
                Lifecycle.Event event = entry.getValue();
                List<MethodReference> methodReferences = mEventToHandlers.get(event);
                if (methodReferences == null) {
                    methodReferences = new ArrayList<>();
                    mEventToHandlers.put(event, methodReferences);
                }
                methodReferences.add(entry.getKey());
            }
        }
        ...
    }
```
我们看到CallbackInfo中有两个Map，一个是我们刚刚才见过的**mHandlerToEvent**，它以MethodReference为Key（即我们创建的LifecycleObserver的方法），以Lifecycle.Event为value。另一个**mEventToHandlers**，它是Lifecycle.Event为Key，value为MethodReferencede的集合。

在CallbackInfo的构造方法中，创建了mEventToHandlers，我们看到，mEventToHandlers就是相同的生命周期事件Lifecycle.Event，对应的所有的方法。


得到CallbackInfo后，我们再看`ReflectiveGenericLifecycleObserver`中的onStateChanged方法，`ReflectiveGenericLifecycleObserver`继承LifecycleEventObserver，当有生命周期事件时，会通知到它的onStateChanged方法，我们看到这里直接调用了CallbackInfo.invokeCallbacks(source, event, mWrapped):


```java
    void invokeCallbacks(LifecycleOwner source, Lifecycle.Event event, Object target) {
        invokeMethodsForEvent(mEventToHandlers.get(event), source, event, target);
        invokeMethodsForEvent(mEventToHandlers.get(Lifecycle.Event.ON_ANY), source, event,target);
    }

    private static void invokeMethodsForEvent(List<MethodReference> handlers,
            LifecycleOwner source, Lifecycle.Event event, Object mWrapped) {
        if (handlers != null) {
            for (int i = handlers.size() - 1; i >= 0; i--) {
                handlers.get(i).invokeCallback(source, event, mWrapped);
            }
        }
    }
```

我们看到，这里就是根据传入的Event，得到MethodReference，调用它的invokeCallback，任何事件都会对ON_ANY事件做处理


```java
  void invokeCallback(LifecycleOwner source, Lifecycle.Event event, Object target) {
            //noinspection TryWithIdenticalCatches
            try {
                switch (mCallType) {
                    case CALL_TYPE_NO_ARG:
                        mMethod.invoke(target);
                        break;
                    case CALL_TYPE_PROVIDER:
                        mMethod.invoke(target, source);
                        break;
                    case CALL_TYPE_PROVIDER_WITH_EVENT:
                        mMethod.invoke(target, source, event);
                        break;
                }
            } catch (InvocationTargetException e) {
                throw new RuntimeException("Failed to call observer method", e.getCause());
            } catch (IllegalAccessException e) {
                throw new RuntimeException(e);
            }
        }

```
invokeCallback根据方法的类型（即方法的参数个数），调用Method的不同的方法。

##### 3.1.1.2 小总结

到这里，我们知道了LifecycleRegistry的addObserver(LifecycleObserver observer)会将传入的LifecycleObserver包装成一个ObserverWithState，这个ObserverWithState持有一个State表示当前状态和一个LifecycleEventObserver。

LifecycleEventObserver是事件的真正的接收者，LifecycleEventObserver是一个接口，它的实现类会持有我们传入的LifecycleObserver。

若我们创建的LifecycleObserver是方法注解的方式，那么ObserverWithState中的LifecycleEventObserver则是`ReflectiveGenericLifecycleObserver`，若我们直接实现LifecycleEventObserver，则ObserverWithState中的LifecycleEventObserver就是我们自己实现的。

ReflectiveGenericLifecycleObserver通过注解找到我们自定义的方法，并缓存方法，当调用时，则通过反射调用响应的方法。

#### 3.1.2 记录ObserverWithState以及杂

```java
   @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        ...
        //2.以LifecycleObserver为key，ObserverWithState为value传入map中记录
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);
        //如果已经添加过，则直接返回
        if (previous != null) {
            return;
        }
        //mLifecycleOwner是WeakReference，持有Activity
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) { 
            return;
        }
        //判断是否代码重入      这个表示正在添加Observer         这个表示正在分发事件
        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        ...
    }
```

得到ObserverWithState后，将它加入到**mObserverMap**中，并且判断是否添加过，以及判断是否LifecycleOwner被回收了。

#### 3.1.3 计算目标State，并更新Observer

```java
    @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
		...
        //3.计算此时的目标State，并将新的observer更新到这个状态
        State targetState = calculateTargetState(observer); 
        //mAddingObserverCounter的值加1
        mAddingObserverCounter++;
        //这个循环的目的是将我们新建的ObserverWithState从初始State（DESTROYED或者INITIALIZED）
        //更新到目前的State
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            //在这个循环的过程中，或许LifecycleOwner（即Activity或Fragment）又改变了状态
            //所以重新计算
            targetState = calculateTargetState(observer);
        }

        ...
    }
```

由于新创建的ObserverWithState中的状态为INITIALIZED或者DESTROYED，很明显，假设我在Activity的onResume中添加一个自定义的LiffecycleObserver，那么真正的State应该为RESUME，所以在添加ObserverWithState后，必须要更新它的状态。

##### 3.1.3.1 计算目标State

```java
    private State calculateTargetState(LifecycleObserver observer) {
        //得到之前的Observer
        Entry<LifecycleObserver, ObserverWithState> previous = mObserverMap.ceil(observer);
		//得到之前的Observer的State
        State siblingState = previous != null ? previous.getValue().mState : null;
        //在改变状态时，会在mParentStates这个List中记录要改变的State，多线程环境下，
        //mParentStates不为空，说明有别的线程正在改变Observer的State，这里记录下其他线程要改变的State
        State parentState = !mParentStates.isEmpty() ? mParentStates.get(mParentStates.size() - 1)
                : null;
        //从当前mState，兄弟State和parentState，得到一个最小的State
        return min(min(mState, siblingState), parentState);
    }

```

很明显，这里计算出来的TargetState，并不是最终的State，而是最小的State。

##### 3.1.3.2 更新LifecycleObserver的State

```java
       while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            //这里就是将当前的Observer的State记录在mParentStates中
            pushParentState(statefulObserver.mState);
            //改变Observer的状态，并分发事件
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            //这里删除掉记录的State
            popParentState();
            //在这个循环的过程中，或许LifecycleOwner（即Activity或Fragment）又改变了状态
            //所以重新计算
            targetState = calculateTargetState(observer);
       }
```

我们不能一下子就更新状态到目标状态，因为需要触发每个生命周期方法，所以必须循环一遍遍比较，并分发事件改变状态

#### 3.1.4 同步多个Observer

```java
    @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
		...
   
        if (!isReentrance) {
            sync();
        }
        mAddingObserverCounter--;
    }
```



这里我们可以看到，当我们添加一个LifecycleObserver时，会立即更新这个Observer的状态为当前组件的状态，在更新的过程中，设置mAddingObserverCounter的值+1，更新状态结束后-1。

### 3.2 LifecyclerOwner通知Lifecycle

LifecycleOwner指的就是Activity、Fragment，而Lifecycle则指它们内部持有的LifecycleRegistry

在Android官网中[自定义LifecycleOwner](https://developer.android.google.cn/topic/libraries/architecture/lifecycle#implementing-lco)中，展示了**LifecycleOwner**如何将生命周期时间通知**LifecycleRegistry**

```kotlin
    class MyActivity : Activity(), LifecycleOwner {

        private lateinit var lifecycleRegistry: LifecycleRegistry

        override fun onCreate(savedInstanceState: Bundle?) {
            super.onCreate(savedInstanceState)

            lifecycleRegistry = LifecycleRegistry(this)
            lifecycleRegistry.markState(Lifecycle.State.CREATED)
        }

        public override fun onStart() {
            super.onStart()
            lifecycleRegistry.markState(Lifecycle.State.STARTED)
        }

        override fun getLifecycle(): Lifecycle {
            return lifecycleRegistry
        }
    }
    
```

LifecycleOwner通过在生命周期方法中调用LifecycleRegistry的`markState`函数将当前的生命周期状态通知给LifecycleRegistry。在onCreate中传入`Lifecycle.State.CREATED`在onStart中传入Lifecycle.State.STARTED

```java
    @Deprecated
    @MainThread
    public void markState(@NonNull State state) {
        setCurrentState(state);
    }

    @MainThread
    public void setCurrentState(@NonNull State state) {
        moveToState(state);
    }
    
    private void moveToState(State next) {
        //当前状态和要设置的状态一致，直接返回
        if (mState == next) {
            return;
        }
        //1.修改状态为最新的状态
        mState = next;
        //3.mHandlingEvent为true，表示正在分发事件设置mNewEventOccurred = true
        //3.mAddingObserverCounter != 0，表示正在添加观察者，添加观察者需要同步它的状态为当前的状态
        //mNewEventOccurred = true表示有新的事件发生，我们会在sync中看到它的作用
        if (mHandlingEvent || mAddingObserverCounter != 0) {
            mNewEventOccurred = true;
            // we will figure out what to do on upper level.
            return;
        }
        //2.mHandlingEvent = true表示正在分发事件
        mHandlingEvent = true;
        sync();
        //4.mHandlingEvent = false表示正在分发事件
        mHandlingEvent = false;
    }

```

我们看到`markState(State state)`最终会调用moveToState(State next)方法，这个方法中会修改LifecycleRegistry的当前状态State，然后调用`sync()`方法分发事件。我们看到这个方法中有几个标志位：

- mHandlingEvent：true 表示正在处理事件中，即正在分发事件
- mNewEventOccurred：true 表示在处理事件时，状态又改变了
- mAddingObserverCounter：我们会在下一篇中提到它，我们可以理解为`mAddingObserverCounter`的数量表示正在添加Observer的数量，当添加完一个Observer`mAddingObserverCounter`会减1。

LifecycleRegistry借助这三个标志来正确的处理生命周期事件分发逻辑，而事件分发通过`sync()`方法去完成，也表示了这个过程是一个需要同步的过程：

```java
    private void sync() {
        //判断LifecycleOwner还没有被回收
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
                    + "garbage collected. It is too late to change lifecycle state.");
        }
        while (!isSynced()) {
            mNewEventOccurred = false;
             //表示状态向后走，如STARTED到CREATED
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
                backwardPass(lifecycleOwner);
            }
            Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            //表示状态向前走，如CREATED到STARTED
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }
    //表示同步完成，即所有状态都统一
    private boolean isSynced() {
        //没有注册观察者直接返回true
        if (mObserverMap.size() == 0) {
            return true;
        }
        //判断第一个和最后一个是否相同，并且最后一个是否和当前状态相同
        State eldestObserverState = mObserverMap.eldest().getValue().mState;
        State newestObserverState = mObserverMap.newest().getValue().mState;
        return eldestObserverState == newestObserverState && mState == newestObserverState;
    }
```

我们看到，核心的方法就是`forwardPass(lifecycleOwner)`和 `backwardPass(lifecycleOwner)`，我们选择`forwardPass(lifecycleOwner)`来看：

```java
    private void forwardPass(LifecycleOwner lifecycleOwner) {
    
        Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
                mObserverMap.iteratorWithAdditions();
        //遍历mObserverMap
        while (ascendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
            ObserverWithState observer = entry.getValue();
            //循环，直到Observer当前的状态为最新的状态
            while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                pushParentState(observer.mState);
                observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
                popParentState();
            }
        }
    }
```

`forwardPass(LifecycleOwner lifecycleOwner)`中，在`upEvent(observer.mState)`中更新Observer的状态，并通过`observer.dispatchEvent(State state)`分发事件。

