---
title: activity,window,view的学习笔记
date: 2019-09-03 11:05
categories: 
- 源码分析
tags:
- android进阶
description: activity,window,view的学习笔记
---

# activity,window,view三者之间的关系

ActivityThread中performLaunchActivity启动activity，并将window  attach到activity上

```java
activity.attach(appContext, this, getInstrumentation(), r.token,
        r.ident, app, r.intent, r.activityInfo, title, r.parent,
        r.embeddedID, r.lastNonConfigurationInstances, config,
        r.referrer, r.voiceInteractor, window, r.configCallback);
```

在activity中attach方法中会创建window即PhoneWindow

```java
//创建PhoneWindow
mWindow = new PhoneWindow(this, window, activityConfigCallback);
mWindow.setWindowControllerCallback(this);
mWindow.setCallback(this);
mWindow.setOnWindowDismissedCallback(this);
//设置LayoutInflater的Factory2，
mWindow.getLayoutInflater().setPrivateFactory(this);
if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
    mWindow.setSoftInputMode(info.softInputMode);
}
if (info.uiOptions != 0) {
    mWindow.setUiOptions(info.uiOptions);
}
```

并且初始化WindowManager

```java
mWindow.setWindowManager(
        (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
        mToken, mComponent.flattenToString(),
        (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
if (mParent != null) {
    mWindow.setContainer(mParent.getWindow());
}
mWindowManager = mWindow.getWindowManager();
```

当执行handleResumeActivity流程的时候

```java
//获取根view
View decor = r.window.getDecorView();
decor.setVisibility(View.INVISIBLE);
//获取WindowManager实际上是WindowManagerIml内部调用WindowManagerGlobal对象
ViewManager wm = a.getWindowManager();
WindowManager.LayoutParams l = r.window.getAttributes();
a.mDecor = decor;
...
 if (r.mPreserveWindow) {
                a.mWindowAdded = true;
                r.mPreserveWindow = false;
                // Normally the ViewRoot sets up callbacks with the Activity
                // in addView->ViewRootImpl#setView. If we are instead reusing
                // the decor view we have to notify the view root that the
                // callbacks may have changed.
   //使用之前的window，直接替换ViewRootImpl，并notifyChildRebuilt
                ViewRootImpl impl = decor.getViewRootImpl();
                if (impl != null) {
                    impl.notifyChildRebuilt();
                }
            }
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                  //会调用WindowManagerGlobal的addView方法
                    wm.addView(decor, l);
                } else {
                    // The activity will get a callback for this {@link LayoutParams} change
                    // earlier. However, at that time the decor will not be set (this is set
                    // in this method), so no action will be taken. This call ensures the
                    // callback occurs with the decor set.
                    a.onWindowAttributesChanged(l);
                }
            }
```

在WindowManagerGlobal的addView方法中

```java
root = new ViewRootImpl(view.getContext(), display);

view.setLayoutParams(wparams);

mViews.add(view);
mRoots.add(root);
mParams.add(wparams);

// do this last because it fires off messages to start doing things
try {
    root.setView(view, wparams, panelParentView);
} catch (RuntimeException e) {
   ...
}
```

保存添加view的相关信息，并将view设置到ViewRootImpl

```java
// Schedule the first layout -before- adding to the window
// manager, to make sure we do the relayout before receiving
// any other events from the system.
//关键方法
requestLayout();
```

最关键的requestLayout方法似曾相识，这个方法会执行scheduleTraversals方法

```java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

我们看看scheduleTraversals方法中都进行了那些操作，

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
      //mTraversalRunnable中是执行的关键
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

我们看一下TraversalRunnable中执行的doTraversal方法

```java
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
```

那么doTraversal中又做了什么呢？

```java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
...

        performTraversals();
。。。
    }
}
```

performTraversals方法中其实就是执行以下三个方法：

```java
performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
performLayout(lp, mWidth, mHeight);
performDraw（）
```

在performTraversals方法中执行了View的dispatchAttachedToWindow方法，会执行view的onAttachedToWindow方法，因为dispatchAttachedToWindow中也执行了mRunQueue.executeActions(info.mHandler);分发所有的pending runnables

其实在该方法中会执行到mRunQueue.executeActions(info.mHandler);我们对这行代码不熟悉，但是我们平常使用mVIew.post(runnable)的时候会获取view的宽高，可是从源码中我们看到该方法是在上述三个方法之前执行的，那我们是如何获取到宽高的呢？其实原因就是我们的measure，layout，draw三大流程都是通过scheduleTraversals方法向下执行的，其中我们看到

```java
mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
```

其实mChoreographer是通过内部的handler将mTraversalRunnable添加到Messagequeue中的，mTraversalRunnable中则执行了我们的三大测量布局绘制流程，接下来我们看一下mRunQueue.executeActions(info.mHandler);

```java
 public void executeActions(Handler handler) {
        synchronized (this) {
            final HandlerAction[] actions = mActions;
            for (int i = 0, count = mCount; i < count; i++) {
                final HandlerAction handlerAction = actions[i];
                handler.postDelayed(handlerAction.action, handlerAction.delay);
            }
...
        }
    }
```

我们看到他是将待执行的handlerAction添加到主线程的Messagequeue中，这个消息时在绘制流程消息的后边，所以会先执行测量布局绘制流程，然后再执行view.post()的行为，所以在这里是可以拿到测量后的宽高，

在这里有需要注意的点：

view.post在api23一下和api24+有所区别，原因就是在于内部使用的HandlerActionQueue不同，在api23一下使用的是ViewRootImpl.getRunQueue,而在api24即以上使用的是view内部new的HandlerActionQueue，但是ViewRootImpl.getRunQueue会在每一帧刷新的时候执行executeActions，view内部的HandlerActionQueue只会在dispatchAttachedToWindow的时候执行executeActions，所以就会存在在api24及以上的版本如果view没有attach是不会执行对应的runnable的，

