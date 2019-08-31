---
title: threadlocal的学习笔记
date: 2019-08-31 11:05
categories: 
- 源码分析
tags:
- android进阶
description: threadlocal的学习笔记
---



# ThreadLoca源码分析

保证对象在单一线程中的唯一性

> 详细分析参考

[ThreadLocal内存泄漏的原因](https://www.jianshu.com/p/dde92ec37bd1)

[基础讲解](https://www.jianshu.com/p/30ee77732843)

> ThreadLocal get方法中的处理逻辑

```java
 public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

> getMap方法通过Thread中的ThreadLocalMap来返回

```java
ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

> 如果map对象为空或者从map中没有找到对应的值，会调用setInitialValue方法

```java
private T setInitialValue() {
  //initialValue默认返回null
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```

此时又会调用getMap方法，如果不为空，则会将value 存入到ThreadLocalMap中，否则会创建ThreadLocalMap

```java
void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

此时会创建ThreadLocalMap并将对应的值存入

> ThreadLocal中的set方法处理逻辑

```java
 public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

## 

##ThreadLocalMap

ThreadLocalMap是ThreadLocal的静态内部类，内部实现是以ThreadLocal的弱引用对象为key的map结构存储  

> ### 内部构造器

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}

```
> ### 内部Entry

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```



> ### getEntry

```java
private Entry getEntry(ThreadLocal<?> key) {
  //计算数组中存储的索引
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

> ### getEntryAfterMiss,如果在ThreadLocalMap中找不到则会调用该方法

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
          //循环过程中找到对应的Entry
            return e;
        if (k == null)
          //脏数据进行清除
            expungeStaleEntry(i);
        else
          //环形增加索引
            i = nextIndex(i, len);
        e = tab[i];
    }
  //没有找到
    return null;
}
```

> ### expungeStaleEntry：废弃掉

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

1. 清理当前脏entry，即将其value引用置为null，并且将table[staleSlot]也置为null。value置为null后该value域变为不可达，在下一次gc的时候就会被回收掉，同时table[staleSlot]为null后以便于存放新的entry;
2. 从当前staleSlot位置向后环形（nextIndex）继续搜索，直到遇到哈希桶（tab[i]）为null的时候退出；
3. 若在搜索过程再次遇到脏entry，继续将其清除。

也就是说该方法，**清理掉当前脏entry后，并没有闲下来继续向后搜索，若再次遇到脏entry继续将其清理，直到哈希桶（table[i]）为null时退出**。因此方法执行完的结果为 **从当前脏entry（staleSlot）位到返回的i位，这中间所有的entry不是脏entry**。为什么是遇到null退出呢？原因是存在脏entry的前提条件是 **当前哈希桶（table[i]）不为null**,只是该entry的key域为null。如果遇到哈希桶为null,很显然它连成为脏entry的前提条件都不具备。

> ### setEntry

```java
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
//碰到hash冲突，向后环形查找
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
//如果已经存在对应的key直接更新value
        if (k == key) {
            e.value = value;
            return;
        }
//如果存在脏数据，则会调用replaceStaleEntry
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
//可以直接插入元素
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

> ### replaceStaleEntry

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    // Back up to check for prior stale entry in current run.
    // We clean out whole runs at a time to avoid continual
    // incremental rehashing due to garbage collector freeing
    // up refs in bunches (i.e., whenever the collector runs).
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

    // Find either the key or trailing null slot of run, whichever
    // occurs first
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // If we find key, then we need to swap it
        // with the stale entry to maintain hash table order.
        // The newly stale slot, or any other stale slot
        // encountered above it, can then be sent to expungeStaleEntry
        // to remove or rehash all of the other entries in run.
        if (k == key) {
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // Start expunge at preceding stale entry if it exists
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // If we didn't find stale entry on backward scan, the
        // first stale entry seen while scanning for key is the
        // first still present in the run.
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // If key not found, put new entry in stale slot
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // If there are any other stale entries in run, expunge them
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

> ### remove

```java
/**
 * Remove the entry for key.
 */
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```

> ### cleanSomeSlots:清除脏entry

1. 从当前位置i处（位于i处的entry一定不是脏entry）为起点在初始小范围（log2(n)，n为哈希表已插入entry的个数size）开始向后搜索脏entry，若在整个搜索过程没有脏entry，方法结束退出
2. 如果在搜索过程中遇到脏entryt通过expungeStaleEntry方法清理掉当前脏entry，并且该方法会返回下一个哈希桶(table[i])为null的索引位置为i。这时重新令搜索起点为索引位置i，n为哈希表的长度len，再次扩大搜索范围为log2(n')继续搜索。

```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```

![](https://upload-images.jianshu.io/upload_images/3044285-e750ddafba97ec6f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 如图当前n等于hash表的size即n=10，i=1,在第一趟搜索过程中通过nextIndex,i指向了索引为2的位置，此时table[2]为null，说明第一趟未发现脏entry,则第一趟结束进行第二趟的搜索。
2. 第二趟所搜先通过nextIndex方法，索引由2的位置变成了i=3,当前table[3]!=null但是该entry的key为null，说明找到了一个脏entry，**先将n置为哈希表的长度len,然后继续调用expungeStaleEntry方法**，该方法会将当前索引为3的脏entry给清除掉（令value为null，并且table[3]也为null）,但是**该方法可不想偷懒，它会继续往后环形搜索**，往后会发现索引为4,5的位置的entry同样为脏entry，索引为6的位置的entry不是脏entry保持不变，直至i=7的时候此处table[7]位null，该方法就以i=7返回。至此，第二趟搜索结束；
3. 由于在第二趟搜索中发现脏entry，n增大为数组的长度len，因此扩大搜索范围（增大循环次数）继续向后环形搜索；
4. 直到在整个搜索范围里都未发现脏entry，cleanSomeSlot方法执行结束退出。

> ### 为什么使用弱引用？

通过threadLocal,threadLocalMap,entry的引用关系看起来threadLocal存在内存泄漏的问题似乎是因为threadLocal是被弱引用修饰的。那为什么要使用弱引用呢？

	> 如果使用强引用

假设threadLocal使用的是强引用，在业务代码中执行`threadLocalInstance==null`操作，以清理掉threadLocal实例的目的，但是因为threadLocalMap的Entry强引用threadLocal，因此在gc的时候进行可达性分析，threadLocal依然可达，对threadLocal并不会进行垃圾回收，这样就无法真正达到业务逻辑的目的，出现逻辑错误

> 使用弱引用

假设Entry弱引用threadLocal，尽管会出现内存泄漏的问题，但是在threadLocal的生命周期里（set,getEntry,remove）里，都会针对key为null的脏entry进行处理。

> ###  Thread.exit()当线程退出时会执行exit方法：
```java
private void exit() {
    if (group != null) {
        group.threadTerminated(this);
        group = null;
    }
    /* Aggressively null out all reference fields: see bug 4006245 */
    target = null;
    /* Speed the release of some of these resources */
    threadLocals = null;
    inheritableThreadLocals = null;
    inheritedAccessControlContext = null;
    blocker = null;
    uncaughtExceptionHandler = null;
}
```

> ### ThreadLocal使用完记得remove

1. 每次使用完ThreadLocal，都调用它的remove()方法，清除数据。
2. 在使用线程池的情况下，没有及时清理ThreadLocal，不仅是内存泄漏的问题，更严重的是可能导致业务逻辑出现问题。所以，使用ThreadLocal就跟加锁完要解锁一样，用完就清理。