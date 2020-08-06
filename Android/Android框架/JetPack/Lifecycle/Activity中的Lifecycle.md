我们之前说过，LifecycleOwner是一个接口，实现该接口，就表示类具有Lifecycle，即这个类是有生命周期的，那么自然，Activity中也实现这个接口
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

那么在ComponentActivity中，是否就是在OnCreate()、onResume()等生命周期方法中调用`markState(State state)`呢？在我们看过源码后，发现并没有。那么ComponentActivity是怎么通知我们添加的Observer呢？我们看到它的onCreate()方法中：


```java
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ReportFragment.injectIfNeededIn(this);
    }

```
这里调用了`ReportFragment.injectIfNeededIn(this);`，我们看到这个ReportFragment：


```java
    public static void injectIfNeededIn(Activity activity) {
        //向当前Activity加入ReportFragment实例
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            //这个方法会立即提交事件
            manager.executePendingTransactions();
        }
    }

```

我们看到，我们每个Activity中，都存在一个ReportFragment，它是一个没有视图的Fragment，它的生命周期方法中会分发Lifecycle事件，如：


```java
    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        dispatchCreate(mProcessListener);
        dispatch(Lifecycle.Event.ON_CREATE);
    }

    @Override
    public void onStart() {
        super.onStart();
        dispatchStart(mProcessListener);
        dispatch(Lifecycle.Event.ON_START);
    }

    @Override
    public void onResume() {
        super.onResume();
        dispatchResume(mProcessListener);
        dispatch(Lifecycle.Event.ON_RESUME);
    }

    @Override
    public void onPause() {
        super.onPause();
        dispatch(Lifecycle.Event.ON_PAUSE);
    }

    @Override
    public void onStop() {
        super.onStop();
        dispatch(Lifecycle.Event.ON_STOP);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        dispatch(Lifecycle.Event.ON_DESTROY);
        // just want to be sure that we won't leak reference to an activity
        mProcessListener = null;
    }

```



```java
    private void dispatch(Lifecycle.Event event) {
        Activity activity = getActivity();
        //LifecycleRegistryOwner是弃用的类
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                //调用handleLifecycleEvent
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }

```

dispathc调用`LifecycleRegistry.handleLifecycleEvent(event)`。


```java
    public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        //根据生命周期的事件，得到State
        State next = getStateAfter(event);
        moveToState(next);
    }

    static State getStateAfter(Event event) {
        switch (event) {
            case ON_CREATE:
            case ON_STOP:
                return CREATED;
            case ON_START:
            case ON_PAUSE:
                return STARTED;
            case ON_RESUME:
                return RESUMED;
            case ON_DESTROY:
                return DESTROYED;
            case ON_ANY:
                break;
        }
        throw new IllegalArgumentException("Unexpected event value " + event);
    }
```

`handleLifecycleEvent`中调用了moveToState，maskState也是调用该方法实现的。

Lifecycle中的State与我们平常接触的生命周期不同，在Lifecycle中，只有**INITIALIZED**、**DESTROYED**、**CREATED**、**STARTED**、**RESUMED**，例如当Activity的生命周期从onStart()到onResume()时，State的状态是从STARTED到RESUME，当从onResume()走到onPause()，那么State的状态就是从RESUME回到了STARTED：

![image](https://developer.android.google.cn/images/topic/libraries/architecture/lifecycle-states.svg)

所以在Activity的生命周期回调中，要先在`handleLifecycleEvent(@NonNull Lifecycle.Event event)`中调用`getStateAfter(Event event)`得到正确的State，然后再通过moveToState(State state)改变状态以及分发事件。


那么为什么Android要在ReportFragment中处理事件的分发，而不是直接在Activity中呢？我想到主要是由于用户自己对生命周期方法的重写，导致不能执行。而用Fragment正好可以在相应的生命周期方法完成效果。

**这里有个注意的地方，我们看到ReportFragment的injectIfNeededIn(Activity activity)方法中，添加Fragment的是android.app.FragmentManager**，它是非兼容版本的FragmentManager，这个版本的FragmentManager的生命周期事件是：


```
Activity_onCreate
Fragment_onAttach
Fragment_onCreate
Fragment_onCreateView
Fragment_onActivityCreate
Activity_onStart
Fragment_onStart
Activity_onResume
Fragment_onResume
```

如果使用androidx包下的FragmentManager，那么会先调用Fragment的onStart，然后调用Activity的onStart。