在ViewRootImpl的setView方法中，不仅设置了DecorView的Parent为ViewRootImpl，还声明了很多InputStage来处理事件的输入。当有点击事件时，会调用View的dispatchPointerEvent方法，然后调用dispatchTouchEvent方法。这里的View就是DecorView。

在DecorView的dispatchTouchEvent方法中，会将事件MotionEvent传递给Window.Callback的dispatchTouchEvent方法,而Activity实现了Window.Callback接口，在Activity的dispatchTouchEvent方法中，又将事件传递给Window的superDispatchTouchEvent，Window的唯一实现类PhoneWindow又会传给DecorView，DecorView最终会调用superDispatchTouchEvent将事件转交给父类，即ViewGroup的dispatchTouchEvent

ViewGroup重写了dispatchTouchEvent，如果允许拦截事件，会调用onInterceptTouchEvent，若onInterceptTouchEvent返回true表示拦截，就会将事件传递给父类的dispatchTouchEvent中，即View的dispatchTouchEvent。若onInterceptTouchEvent返回false不拦截，就会遍历子View，它判断子View是否接受事件以及是否落点在子View的范围内，如果满足条件，则将事件传递给子View的dispatchTouchEvent。

View的dispatchTouchEvent首先判断OnTouchListener.onTouch方法，如果返回true，事件就被消费了，如果返回false，则继续向下，调用onTouchEvent方法，若onTouchEvent返回true，表示事件被消费，若返回false，表示不消费

由于View或者ViewGroup的dispatchTouchEvent方法都是有父类调用的，当返回flase时，表示子View不消费，所有事件最终由Activity消费，调用Activity的onTouchEvent，它默认为false，即事件最终不消费。

