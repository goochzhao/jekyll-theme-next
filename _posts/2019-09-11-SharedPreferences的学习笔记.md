---
title: SharedPreferences的学习笔记
date: 2019-09-11 11:05
categories: 
- 源码解析
tags:
- android进阶
description: SharedPreferences的学习笔记
---

# SharedPreferences源码解析

>  SharedPreferences的使用

```java
SharedPreferences sharedPref = context.getSharedPreferences(
        getString(R.string.preference_file_key), Context.MODE_PRIVATE);

```

mode指定为MODE_PRIVATE，则该配置文件只能被自己的应用程序访问。 
mode指定为MODE_WORLD_READABLE，则该配置文件除了自己访问外还可以被其它应该程序读取。 
mode指定为MODE_WORLD_WRITEABLE，则该配置文件除了自己访问外还可以被其它应该程序读取和写入

```java
SharedPreferences.Editor editor = sharedPref.edit();
editor.putInt(getString(R.string.saved_high_score), newHighScore);
editor.commit();
long highScore = sharedPref.getInt(getString(R.string.saved_high_score), default);
```

通过创建`SharedPreferences.Editor` 来操作内容的读取和写入

**访问其他应用的SharedPreferences**

如果应用B要读写访问A应用中的Preference前提条件,A应用中该preference创建时指定了Context.MODE_WORLD_READABLE或者Context.MODE_WORLD_WRITEABLE权限，代表其他的应用能访问读取或者写入。 

1.在B中创建一个指向A应用的Context

```java
Context otherAppsContext = createPackageContext("A应用的包名",				Context.CONTEXT_IGNORE_SECURITY);
```

2.再通过context获取到SharedPreferences实体

```java
SharedPreferences sharedPreferences = otherAppsContext.getSharedPreferences("SharedPreferences的文件    名", Context.MODE_WORLD_READABLE);
String name = sharedPreferences.getString("key", "");
```

也可以以读取xml文件方式直接访问其他应用preference对应的xml文件，如：

```java
File xmlFile = new File(“/data/data/<package name>/shared_prefs/itcast.xml”);
//<package name>应替换成应用的包名。
```

> 源码解析

首先在`ContextImpl` 中`getSharedPreferences` 会调用获取`SharedPreferences` 实例：

```java
public SharedPreferences getSharedPreferences(String name, int mode) {
        // At least one application in the world actually passes in a null
        // name.  This happened to work because when we generated the file name
        // we would stringify it to "null.xml".  Nice.
        if (mPackageInfo.getApplicationInfo().targetSdkVersion <
                Build.VERSION_CODES.KITKAT) {
            if (name == null) {
                name = "null";
            }
        }

        File file;
        synchronized (ContextImpl.class) {
            if (mSharedPrefsPaths == null) {
                mSharedPrefsPaths = new ArrayMap<>();
            }
            file = mSharedPrefsPaths.get(name);
            if (file == null) {
              //通过文件名获取文件
                file = getSharedPreferencesPath(name);
                mSharedPrefsPaths.put(name, file);
            }
        }
  				//
        return getSharedPreferences(file, mode);
    }
```

接下来我们看一下`getSharedPreferences` 中是如何生成`SharedPreferences`:

```java
  @Override
    public SharedPreferences getSharedPreferences(File file, int mode) {
        SharedPreferencesImpl sp;
        synchronized (ContextImpl.class) {
        
            final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
            sp = cache.get(file);
            if (sp == null) {
              //检查模式是否支持，不支持会抛异常
                checkMode(mode);
                if (getApplicationInfo().targetSdkVersion >= android.os.Build.VERSION_CODES.O) {
                  //如果是Android8.0以上，进行文件加密检查
                    if (isCredentialProtectedStorage()
                            && !getSystemService(UserManager.class)
                                    .isUserUnlockingOrUnlocked(UserHandle.myUserId())) {
                        throw new IllegalStateException("SharedPreferences in credential encrypted "
                                + "storage are not available until after user is unlocked");
                    }
                }
                sp = new SharedPreferencesImpl(file, mode);
                cache.put(file, sp);
                return sp;
            }
        }
        if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
            getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
            // If somebody else (some other process) changed the prefs
            // file behind our back, we reload it.  This has been the
            // historical (if undocumented) behavior.
            sp.startReloadIfChangedUnexpectedly();
        }
        return sp;
    }
```

我们来看一下`checkMode`方法是如何检查的：

```java
  private void checkMode(int mode) {
        if (getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.N) {
            if ((mode & MODE_WORLD_READABLE) != 0) {
                throw new SecurityException("MODE_WORLD_READABLE no longer supported");
            }
            if ((mode & MODE_WORLD_WRITEABLE) != 0) {
                throw new SecurityException("MODE_WORLD_WRITEABLE no longer supported");
            }
        }
    }
```

由上面代码可知，在目标SDK版本高于23是不支持MODE_WORLD_READABLE和MODE_WORLD_WRITEABLE这两种模式的

```java
final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
            sp = cache.get(file);
```

```java
 private ArrayMap<File, SharedPreferencesImpl> getSharedPreferencesCacheLocked() {
        if (sSharedPrefsCache == null) {
            sSharedPrefsCache = new ArrayMap<>();
        }

        final String packageName = getPackageName();
        ArrayMap<File, SharedPreferencesImpl> packagePrefs = sSharedPrefsCache.get(packageName);
        if (packagePrefs == null) {
            packagePrefs = new ArrayMap<>();
            sSharedPrefsCache.put(packageName, packagePrefs);
        }

        return packagePrefs;
    }
```

sSharedPrefsCache是`private static ArrayMap<String, ArrayMap<File, SharedPreferencesImpl>> sSharedPrefsCache;` 以包名为key的map，整体逻辑就是首先通过缓存中去查找对应的sp，找不到会直接创建并放入到缓存中，这就是获取SharedPreferences对象逻辑。

接下来我们看一下SharedPreferencesImpl

```java
 SharedPreferencesImpl(File file, int mode) {
        mFile = file;
   			//进行文件备份
        mBackupFile = makeBackupFile(file);
        mMode = mode;
        mLoaded = false;
        mMap = null;
        mThrowable = null;
   			//
        startLoadFromDisk();
    }
```

```java
	 private void startLoadFromDisk() {
        synchronized (mLock) {
            mLoaded = false;
        }
        new Thread("SharedPreferencesImpl-load") {
            public void run() {
                loadFromDisk();
            }
        }.start();
    }
```

构造方法主要进行了建立备份文件实例，加载文件到内存的操作，在startLoadFromDisk()方法中，通过同步枷锁，启动线程将文件内容加入到内存，这里提供了一层线程安全保障

```java
  private void loadFromDisk() {
        synchronized (mLock) {
            if (mLoaded) {
                return;
            }
            if (mBackupFile.exists()) {
                mFile.delete();
                mBackupFile.renameTo(mFile);
            }
        }

        // Debugging
        if (mFile.exists() && !mFile.canRead()) {
            Log.w(TAG, "Attempt to read preferences file " + mFile + " without permission");
        }

        Map<String, Object> map = null;
        StructStat stat = null;
        Throwable thrown = null;
        try {
            stat = Os.stat(mFile.getPath());
            if (mFile.canRead()) {
                BufferedInputStream str = null;
                try {
                    str = new BufferedInputStream(
                            new FileInputStream(mFile), 16 * 1024);
                    map = (Map<String, Object>) XmlUtils.readMapXml(str);
                } catch (Exception e) {
                    Log.w(TAG, "Cannot read " + mFile.getAbsolutePath(), e);
                } finally {
                    IoUtils.closeQuietly(str);
                }
            }
        } catch (ErrnoException e) {
            // An errno exception means the stat failed. Treat as empty/non-existing by
            // ignoring.
        } catch (Throwable t) {
            thrown = t;
        }

        synchronized (mLock) {
            mLoaded = true;
            mThrowable = thrown;

            // It's important that we always signal waiters, even if we'll make
            // them fail with an exception. The try-finally is pretty wide, but
            // better safe than sorry.
            try {
                if (thrown == null) {
                    if (map != null) {
                        mMap = map;
                        mStatTimestamp = stat.st_mtim;
                        mStatSize = stat.st_size;
                    } else {
                        mMap = new HashMap<>();
                    }
                }
                // In case of a thrown exception, we retain the old map. That allows
                // any open editors to commit and store updates.
            } catch (Throwable t) {
                mThrowable = t;
            } finally {
              	//通知所有线程等待的操作可以继续
                mLock.notifyAll();
            }
        }
    }
```

首先是加锁判断当前状态，如果已经加载完成，就直接返回，否则会重新设置备份的文件对象。然后使用XmlUtils读取文件内容，并保存到Map对象mMap中，通知所有正在等待的客户，文件读取完成，可以进行键值对的读写操作了。这里需要注意：系统会一次性把SP文件内容读入到内存并保存在Map对象中。这就是SP文件内容的加载过程

> 读取元素

下面看一下如何获取元素

```java
@Override
    public int getInt(String key, int defValue) {
        synchronized (mLock) {
            awaitLoadedLocked();
            Integer v = (Integer)mMap.get(key);
            return v != null ? v : defValue;
        }
    }
```

awaitLoadedLocked方法主要是进行了锁等待操作,如果mLoaded为false就会执行线程等待操作，对应的情况就是SP文件还未读取，如果文件内容没有读取完成，getInt()方法会一直阻塞

```java
 private void awaitLoadedLocked() {
        if (!mLoaded) {
            // Raise an explicit StrictMode onReadFromDisk for this
            // thread, since the real read will be in a different
            // thread and otherwise ignored by StrictMode.
            BlockGuard.getThreadPolicy().onReadFromDisk();
        }
        while (!mLoaded) {
            try {
                mLock.wait();
            } catch (InterruptedException unused) {
            }
        }
        if (mThrowable != null) {
            throw new IllegalStateException(mThrowable);
        }
    }
```

> 如何写入元素

SP的写入操作是由EditorImpl具体实现的。对SP中键值对的读写操作，分两步，第一步是由EditorImpl直接更改mMap的值；第二步，将mMap中的键值对写入到文件中保存。这里需要重点关注两个方法：apply()和commit()，它们的实现如下

EditorImpl中有以下

我们来看一下他是如何获取EditorImpl对象的：

```java
 public Editor edit() {
		// TODO: remove the need to call awaitLoadedLocked() when
    // requesting an editor.  will require some work on the
    // Editor, but then we should be able to do:
    //
    //      context.getSharedPreferences(..).edit().putString(..).apply()
    //
    // ... all without blocking.
    synchronized (mLock) {
        awaitLoadedLocked();
    }

    return new EditorImpl();
}
```

这里只是直接new 了一个EditorImpl对象，

```java
private final Map<String, Object> mModified = new HashMap<>();
```

该集合用于存放提交的键值对，

```java
public Editor putString(String key, @Nullable String value) {
    synchronized (mEditorLock) {
        mModified.put(key, value);
        return this;
    }
}
```

我们每次操作都会将键值对放入对应的HashMap中，用于提交

```java
public void apply() {
    final long startTime = System.currentTimeMillis();

    final MemoryCommitResult mcr = commitToMemory();
    final Runnable awaitCommit = new Runnable() {
            @Override
            public void run() {
                try {
                    mcr.writtenToDiskLatch.await();
                } catch (InterruptedException ignored) {
                }

                if (DEBUG && mcr.wasWritten) {
                    Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                            + " applied after " + (System.currentTimeMillis() - startTime)
                            + " ms");
                }
            }
        };

    QueuedWork.addFinisher(awaitCommit);

    Runnable postWriteRunnable = new Runnable() {
            @Override
            public void run() {
                awaitCommit.run();
                QueuedWork.removeFinisher(awaitCommit);
            }
        };

    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

    // Okay to notify the listeners before it's hit disk
    // because the listeners should always get the same
    // SharedPreferences instance back, which has the
    // changes reflected in memory.
    notifyListeners(mcr);
}
```



```java
public boolean commit() {
    long startTime = 0;

    if (DEBUG) {
        startTime = System.currentTimeMillis();
    }

    MemoryCommitResult mcr = commitToMemory();

    SharedPreferencesImpl.this.enqueueDiskWrite(
        mcr, null /* sync write on this thread okay */);
    try {
      	//通过CountDownLatch来等待将内容同步写入到文件中
        mcr.writtenToDiskLatch.await();
    } catch (InterruptedException e) {
        return false;
    } finally {
        if (DEBUG) {
            Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                    + " committed after " + (System.currentTimeMillis() - startTime)
                    + " ms");
        }
    }
    notifyListeners(mcr);
    return mcr.writeToDiskResult;
}
```

提交内容到内存中

```java
private MemoryCommitResult commitToMemory() {
    long memoryStateGeneration;
    List<String> keysModified = null;
    Set<OnSharedPreferenceChangeListener> listeners = null;
    Map<String, Object> mapToWriteToDisk;

    synchronized (SharedPreferencesImpl.this.mLock) {
        // We optimistically don't make a deep copy until
        // a memory commit comes in when we're already
        // writing to disk.
        if (mDiskWritesInFlight > 0) {
            // We can't modify our mMap as a currently
            // in-flight write owns it.  Clone it before
            // modifying it.
            // noinspection unchecked
            mMap = new HashMap<String, Object>(mMap);
        }
        mapToWriteToDisk = mMap;
        mDiskWritesInFlight++;

        boolean hasListeners = mListeners.size() > 0;
        if (hasListeners) {
            keysModified = new ArrayList<String>();
            listeners = new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
        }

        synchronized (mEditorLock) {
            boolean changesMade = false;

            if (mClear) {
                if (!mapToWriteToDisk.isEmpty()) {
                    changesMade = true;
                    mapToWriteToDisk.clear();
                }
                mClear = false;
            }

            for (Map.Entry<String, Object> e : mModified.entrySet()) {
                String k = e.getKey();
                Object v = e.getValue();
                // "this" is the magic value for a removal mutation. In addition,
                // setting a value to "null" for a given key is specified to be
                // equivalent to calling remove on that key.
                if (v == this || v == null) {
                    if (!mapToWriteToDisk.containsKey(k)) {
                        continue;
                    }
                    mapToWriteToDisk.remove(k);
                } else {
                    if (mapToWriteToDisk.containsKey(k)) {
                        Object existingValue = mapToWriteToDisk.get(k);
                        if (existingValue != null && existingValue.equals(v)) {
                            continue;
                        }
                    }
                    mapToWriteToDisk.put(k, v);
                }

                changesMade = true;
                if (hasListeners) {
                    keysModified.add(k);
                }
            }

            mModified.clear();

            if (changesMade) {
                mCurrentMemoryStateGeneration++;
            }

            memoryStateGeneration = mCurrentMemoryStateGeneration;
        }
    }
    return new MemoryCommitResult(memoryStateGeneration, keysModified, listeners,
            mapToWriteToDisk);
}
```

可以看到，commit 和 apply 操作首先执行了 commitToMemory，顾名思义就是提交到内存，返回值是 MemoryCommitResult 类型，里面保存着本次提交的状态。然后 commit 调用 enqueueDiskWrite 会阻塞当前线程，而 apply 通过封装 Runnable 把写磁盘之后的操作传递给 enqueueDiskWrite 方法。

enqueueDiskWrite 方法，参数有 MemoryCommitResult 和 Runnable，mcr 刚才说过，就看这个 Runnable 是干嘛的。在 commit 方法中调用 enqueueDiskWrite 方法是传入的 Runnable 是null，它会在当前线程直接执行写文件的操作，然后返回写入结果。而如果 Runnable 不是 null，那就使用 QueueWork 中的单线程执行。这就是 apply 和 commit 的根本区别：一个同步执行，有返回值；一个异步执行，没有返回值。大多数情况下，我们使用 apply 就够了，这也是官方推荐的做法

```java
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                              final Runnable postWriteRunnable) {
    final boolean isFromSyncCommit = (postWriteRunnable == null);

    final Runnable writeToDiskRunnable = new Runnable() {
            @Override
            public void run() {
                synchronized (mWritingToDiskLock) {
                    writeToFile(mcr, isFromSyncCommit);
                }
                synchronized (mLock) {
                    mDiskWritesInFlight--;
                }
                if (postWriteRunnable != null) {
                    postWriteRunnable.run();
                }
            }
        };

    // Typical #commit() path with fewer allocations, doing a write on
    // the current thread.
    if (isFromSyncCommit) {
        boolean wasEmpty = false;
        synchronized (mLock) {
            wasEmpty = mDiskWritesInFlight == 1;
        }
        if (wasEmpty) {
            writeToDiskRunnable.run();
            return;
        }
    }
		//通过HandlerThread的实现，将apply操作的runnable进行队列执行
    QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
}
```

再看一下具体写文件的代码 writeToFile 方法，关键点在代码中文注释部分。简单说就是备份 → 写入 → 检查 → 善后，这样保证了数据的安全性和稳定性。

```java
private void writeToFile(MemoryCommitResult mcr, boolean isFromSyncCommit) {
   ...

    boolean fileExists = mFile.exists();

    // Rename the current file so it may be used as a backup during the next read
    if (fileExists) {
        boolean needsWrite = false;

        // Only need to write if the disk state is older than this commit
        if (mDiskStateGeneration < mcr.memoryStateGeneration) {
            if (isFromSyncCommit) {
                needsWrite = true;
            } else {
                synchronized (mLock) {
                    // No need to persist intermediate states. Just wait for the latest state to
                    // be persisted.
                    if (mCurrentMemoryStateGeneration == mcr.memoryStateGeneration) {
                        needsWrite = true;
                    }
                }
            }
        }

        if (!needsWrite) {
            mcr.setDiskWriteResult(false, true);
            return;
        }

        boolean backupFileExists = mBackupFile.exists();

        if (DEBUG) {
            backupExistsTime = System.currentTimeMillis();
        }

        if (!backupFileExists) {
            if (!mFile.renameTo(mBackupFile)) {
                Log.e(TAG, "Couldn't rename file " + mFile
                      + " to backup file " + mBackupFile);
                mcr.setDiskWriteResult(false, false);
                return;
            }
        } else {
            mFile.delete();
        }
    }

    // Attempt to write the file, delete the backup and return true as atomically as
    // possible.  If any exception occurs, delete the new file; next time we will restore
    // from the backup.
    try {
        FileOutputStream str = createFileOutputStream(mFile);

        if (DEBUG) {
            outputStreamCreateTime = System.currentTimeMillis();
        }

        if (str == null) {
            mcr.setDiskWriteResult(false, false);
            return;
        }
        XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);

        writeTime = System.currentTimeMillis();

        FileUtils.sync(str);

        fsyncTime = System.currentTimeMillis();

        str.close();
        ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);

        if (DEBUG) {
            setPermTime = System.currentTimeMillis();
        }

        try {
            final StructStat stat = Os.stat(mFile.getPath());
            synchronized (mLock) {
                mStatTimestamp = stat.st_mtim;
                mStatSize = stat.st_size;
            }
        } catch (ErrnoException e) {
            // Do nothing
        }
				//删除备份文件
        // Writing was successful, delete the backup file if there is one.
        mBackupFile.delete();
        mDiskStateGeneration = mcr.memoryStateGeneration;
				//存储磁盘成功，通知countdownlatch释放等待操作
        mcr.setDiskWriteResult(true, true);
        return;
    } catch (XmlPullParserException e) {
        Log.w(TAG, "writeToFile: Got exception:", e);
    } catch (IOException e) {
        Log.w(TAG, "writeToFile: Got exception:", e);
    }
		//存储磁盘失败，删除配置文件
    // Clean up an unsuccessfully written file
    if (mFile.exists()) {
        if (!mFile.delete()) {
            Log.e(TAG, "Couldn't clean up partially-written file " + mFile);
        }
    }
    mcr.setDiskWriteResult(false, false);
}
```



> 是否是线程安全的？

是线程安全的，存取操作都是同步加锁的操作。



> 为什么是进程非安全的？



> 总结

1. 通过 getSharedPreferences 可以获取 SP 实例，从首次初始化到读到数据会存在延迟，因为读文件的操作阻塞调用的线程直到文件读取完毕，如果在主线程调用，可能会对 UI 流畅度造成影响。

2. commit 会在调用者线程同步执行写文件，返回写入结果；apply 将写文件的操作异步执行，没有返回值。可以根据具体情况选择性使用，推荐使用 apply。

3. 虽然支持设置 MODE_MULTI_PROCESS 标志位，但是跨进程共享 SP 存在很多问题，所以不建议使用该模式。

4. ### 勿存储过大value

   - 永远记住，SharedPreferences是一个轻量级的存储系统，不要存过多且复杂的数据，这会带来以下的问题

      

     - 第一次从sp中获取值的时候，有可能阻塞主线程，使界面卡顿、掉帧。
     - 这些key和value会永远存在于内存之中，占用大量内存。

5. 勿存储复杂数据

   SharedPreferences通过xml存储解析，JSON或者HTML格式存放在sp里面的时候，需要转义，这样会带来很多&这种特殊符号，sp在解析碰到这个特殊符号的时候会进行特殊的处理，引发额外的字符串拼接以及函数调用开销。如果数据量大且复杂，严重时可能导频繁GC

6. ### 不要乱edit和apply，尽量批量修改一次提交

   - edit会创建editor对象，每进行一次apply就会创建线程，进行内存和磁盘的同步，千万写类似下面的代码

7. apply和commit对registerOnSharedPreferenceChangeListener的影响

   对于 apply, listener 回调时内存已经完成同步, 但是异步磁盘任务不保证是否完成
   对于 commit, listener 回调时内存和磁盘都已经同步完毕


