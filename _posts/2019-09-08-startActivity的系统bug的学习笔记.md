---
title: startActivity的系统bug的学习笔记
date: 2019-09-08 11:05
categories: 
- 源码学习
tags:
- android进阶
description: startActivity的系统bug学习笔记
---

# startActivity的系统bug

我们知道启动activity是可以通过activity，service，applicationcontext启动，但是他们有哪些不同呢？

```java
 public void startActivity(Intent intent, Bundle options) {
  
        // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
        // generally not allowed, except if the caller specifies the task id the activity should
        // be launched in. A bug was existed between N and O-MR1 which allowed this to work. We
        // maintain this for backwards compatibility.
        final int targetSdkVersion = getApplicationInfo().targetSdkVersion;

        if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
                && (targetSdkVersion < Build.VERSION_CODES.N
                        || targetSdkVersion >= Build.VERSION_CODES.P)
                && (options == null
                        || ActivityOptions.fromBundle(options).getLaunchTaskId() == -1)) {
            throw new AndroidRuntimeException(
                    "Calling startActivity() from outside of an Activity "
                            + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                            + " Is this really what you want?");
        }
       。。。
    }
```

上述源码为Android 28的源码版本：

通过注释信息我们能够得到这么几个信息

1、并不是一定要带NEW_TASK，如果指定了任务栈，也没问题，这一点从判断逻辑中的 ActivityOptions.fromBundle(options).getLaunchTaskId() == -1 即可看出；
2、竟然这个异常在Android N到O上不会抛出，且谷歌指明这是一个Bug，只是为了兼容，保留了这个判断：targetSdkVersion < Build.VERSION_CODES.N || targetSdkVersion >= Build.VERSION_CODES.P ，因此对于targetSDK小于N或大于等于P的应用，就会正常地抛出此异常。

那么我们来看一下android26  O版本的源码：到底问题出在哪个地方

```java
 // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
        // generally not allowed, except if the caller specifies the task id the activity should
        // be launched in.
        if ((intent.getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0
                && options != null && ActivityOptions.fromBundle(options).getLaunchTaskId() == -1) {
            throw new AndroidRuntimeException(
                    "Calling startActivity() from outside of an Activity "
                    + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                    + " Is this really what you want?");
        }
```

通过上边的对比我们发现在判断条件处&&options!=null发生逻辑错误，在android O中如果设置options为null则直接跳过了其他的判断，启动了activity，