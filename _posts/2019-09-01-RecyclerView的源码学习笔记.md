---
title: RecyclerView的源码学习笔记
date: 2019-09-01 22:05
categories: 
- 源码分析
tags:
- android进阶
description: RecyclerView的源码学习笔记
---

# RecyclerView的源码

参考文章：[https://mp.weixin.qq.com/s/S31bHWLtUeR4-sjI-ULoUQ](https://mp.weixin.qq.com/s/S31bHWLtUeR4-sjI-ULoUQ)

[https://mp.weixin.qq.com/s/FiQEa0M93eSi1i4PzW_6Nw](https://mp.weixin.qq.com/s/FiQEa0M93eSi1i4PzW_6Nw)

[https://mp.weixin.qq.com/s/AUEDB--AHy4kLUHnMzjFYg](https://mp.weixin.qq.com/s/AUEDB--AHy4kLUHnMzjFYg)



## RecyclerView的回收机制

```java
  public final class Recycler {
        final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
        ArrayList<ViewHolder> mChangedScrap = null;

        final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();
        RecycledViewPool mRecyclerPool;
        private ViewCacheExtension mViewCacheExtension;
        …… 省略 ……
    }
```

1. mAttachedScrap:我们可以看到这个变量是个存放ViewHolder对象的ArrayList,这一级缓存是没有容量限制的，只要符合条件的我来者不拒，全收了。前面讲两个专业术语的时候提到了Scrap,这个就属于Scrap中的一种，这里的数据是不做修改的，不会重新走Adapter的绑定方法。
2. mChangedScrap:这个变量和上边的mAttachedScrap是一样的，唯一不同的从名字也可以看出来，它存放的是发生了变化的ViewHolder,如果使用到了这里的缓存的ViewHolder是要重新走Adapter的绑定方法的。
3. mCachedViews:这个变量同样是一个存放ViewHolder对象的ArrayList,但是这个不同于上面的两个里面存放的是dettach掉的视图，它里面存放的是已经remove掉的视图，已经和RV分离的关系的视图，但是它里面的ViewHolder依然保存着之前的信息，比如position、和绑定的数据等等。这一级缓存是有容量限制的，默认是2（不同版本API可能会有差异，本文基于API26.1.0）。
4. mRecyclerPool:这个变量呢本身是一个类，跟上面三个都不一样。这里面保存的ViewHolder不仅仅是removed掉的视图，而且是恢复了出厂设置的视图，任何绑定过的痕迹都没有了，想用这里缓存的ViewHolder那是铁定要重新走Adapter的绑定方法了。而且我们知道RV支持多布局，所以这里的缓存是按照itemType来分开存储的，我们来大致的看一下它的结构

```java
   public static class RecycledViewPool {
        private static final int DEFAULT_MAX_SCRAP = 5;
        static class ScrapData {
            ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
            int mMaxScrap = DEFAULT_MAX_SCRAP;
            …… 省略 ……
        }
        SparseArray<ScrapData> mScrap = new SparseArray<>();
        …… 省略后面代码 ……
    }
```

- 首先我们看到一个常量‘DEFAULT_MAX_SCRAP’，这个就是缓存池定义的一个默认的缓存数，当然这个缓存数我们是可以自己设置的。而且这个缓存数量不是指整个缓存池只能缓存这么多，而是每个不同itemType的ViewHolder的缓存数量。
- 接着往下看，我们看到一个静态内部类ScrapData,这里我们只看跟缓存相关的两个变量，先说mMaxScrap，前面的常量赋值给了它，这也就印证了我们前面说的这个缓存数量是对应每一种类型的ViewHolder的。再来看这个mScrapHeap变量，熟悉的一幕又来了，同样是一个缓存ViewHolder对象的ArrayList，它的容量默认是5.
- 最后我们看到mScrap这个变量，它是一个存储我们上面提到的ScrapData类的对象的SparseArray，这样我们这个RecyclerPool就把不同itemType的ViewHolder按类型分类缓存了起来

mViewCacheExtension：这一级缓存是留给开发者自由发挥的，官方并没有默认实现，它本身是null。

缓存层级讲完了。这里提一句，其实还有一层没有提到，因为它不在Recycler这个类中，它在ChildHelper类中，其中有个mHiddenViews,是个缓存被隐藏的ViewHolder的ArrayList。到这里我想大家对这几层缓存心里已经有个数了，但是还远远不够，这么多层缓存是怎么工作的？什么时候用什么缓存？各个缓存之间有没有什么PY交易？如果让你自己写一个LayoutManager你能处理好缓存问题么？就好比垃圾分类后，我们知道每种垃圾桶的定义和功能，ß下面我们一起来看看，这些个缓存都咋用。