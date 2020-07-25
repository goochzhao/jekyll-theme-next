---
title: ViewPager调用adapter的notifyDataSetChange方法无效问题
date: 2018-04-18 19:32
categories:
- Viewpager刷新问题
tags:
- viewpager

description: ViewPager 刷新问题分别使用FragmentPageAdapter和FragmentPageStateAdapter有不同的解决方法
---
## 问题描述
>问题描述：Viewpager+FragmentPageAdapter 由于要重新刷新adapter中的fragment数量，所以我就调用了adapter的notifyDataSetChange()方法，结果就是其中的fragment中的数据竟然错乱了，一步步debug发现数据竟然不是对应的数据源。。。。。 :smile::smile::smile:

正常情况下一般不会去更改adapter中的fragment的数量，只是改变其内容，这不会出现什么问题，但是当你更改数量时，调用notifyDataSetChange()方法时就会出问题，排查问题后发现这里边有坑啊！！！！


----------
## 解决方案
先说解决方案，原理待会再说，主要有两种：

### 如果设置的adapter类型是FragmentPageAdapter

 在自己定义的adapter中重写以下几个方法：
>这是一种复用性比较好的解决方案


```java
@Override
public int getItemPosition(Object object) {
     return PagerAdapter.POSITION_NONE;
}
@Override
 public Object instantiateItem(ViewGroup container, int position) {
    MyFragment f = (MyFragment) super.instantiateItem(container, position);
    String title = mList.get(position);
    f.setTitle(title);
    return f;
 }
@Override
public Fragment getItem(int position) {
      MyFragment f = new MyFragment();
      return f;
  }
```


>还有一种是丧失FragmentPagerAdapter特性的方案


```java
@Override
public int getItemPosition(Object object) {
     return PagerAdapter.POSITION_NONE;
}
public void setFragments(ArrayList fragments) {
   if(this.fragments != null){
      FragmentTransaction ft = fm.beginTransaction();
      for(Fragment f:this.fragments){
        ft.remove(f);
      }
      ft.commit();
      ft=null;
      fm.executePendingTransactions();
   }
  this.fragments = fragments;
  notifyDataSetChanged();
}
```



### 如果设置的adapter是FragmentStatePagerAdapter

这个解决方案就比较简单了,直接在自己写的adapter中重写下列方法即可
```java
@Override
public int getItemPosition(Object object) {
     return PagerAdapter.POSITION_NONE;
}
```


----------

## 原理解析
### 在这里先说一下getItemPosition方法，

因为在viewpager中默认该方法返回的是POSITION_UNCHANGED
```java
/**
     * Called when the host view is attempting to determine if an item's position
     * has changed. Returns {@link #POSITION_UNCHANGED} if the position of the given
     * item has not changed or {@link #POSITION_NONE} if the item is no longer present
     * in the adapter.
     *
     * <p>The default implementation assumes that items will never
     * change position and always returns {@link #POSITION_UNCHANGED}.
     *
     * @param object Object representing an item, previously returned by a call to
     *               {@link #instantiateItem(View, int)}.
     * @return object's new position index from [0, {@link #getCount()}),
     *         {@link #POSITION_UNCHANGED} if the object's position has not changed,
     *         or {@link #POSITION_NONE} if the item is no longer present.
     */
    public int getItemPosition(Object object) {
        return POSITION_UNCHANGED;
    }
```
在 PagerAdapter 中的实现是直接传回 POSITION_UNCHANGED。如果该函数不被重载，则会一直返回 POSITION_UNCHANGED，从而导致 ViewPager.dataSetChanged() 被调用时，认为不必触发 PagerAdapter.instantiateItem()。很多人因为没有重载该函数，而导致调用
 PagerAdapter.notifyDataSetChanged() 后，什么都没有发生。
所以我们要重写该方法并返回值POSITION_NONE
从ViewPager中dataSetChanged方法中可以看到：
```java
 void dataSetChanged() {
        // This method only gets called if our observer is attached, so mAdapter is non-null.

        final int adapterCount = mAdapter.getCount();
        mExpectedAdapterCount = adapterCount;
        boolean needPopulate = mItems.size() < mOffscreenPageLimit * 2 + 1
                && mItems.size() < adapterCount;
        int newCurrItem = mCurItem;

        boolean isUpdating = false;
        for (int i = 0; i < mItems.size(); i++) {
            final ItemInfo ii = mItems.get(i);
            final int newPos = mAdapter.getItemPosition(ii.object);

            if (newPos == PagerAdapter.POSITION_UNCHANGED) {
                continue;
            }

            if (newPos == PagerAdapter.POSITION_NONE) {
                mItems.remove(i);
                i--;

                if (!isUpdating) {
                    mAdapter.startUpdate(this);
                    isUpdating = true;
                }

                mAdapter.destroyItem(this, ii.position, ii.object);
                needPopulate = true;

                if (mCurItem == ii.position) {
                    // Keep the current item in the valid range
                    newCurrItem = Math.max(0, Math.min(mCurItem, adapterCount - 1));
                    needPopulate = true;
                }
                continue;
            }
       。。。省略
    }
```
### 对于FragmentPagerAdapter看源码中的这两个方法即可

```java
 @Override
    public Object instantiateItem(ViewGroup container, int position) {
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }

        final long itemId = getItemId(position);

        // Do we already have this fragment?
        String name = makeFragmentName(container.getId(), itemId);
        Fragment fragment = mFragmentManager.findFragmentByTag(name);
        if (fragment != null) {
            if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
            mCurTransaction.attach(fragment);
        } else {
            fragment = getItem(position);
            if (DEBUG) Log.v(TAG, "Adding item #" + itemId + ": f=" + fragment);
            mCurTransaction.add(container.getId(), fragment,
                    makeFragmentName(container.getId(), itemId));
        }
        if (fragment != mCurrentPrimaryItem) {
            fragment.setMenuVisibility(false);
            fragment.setUserVisibleHint(false);
        }

        return fragment;
    }
  @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }
        if (DEBUG) Log.v(TAG, "Detaching item #" + getItemId(position) + ": f=" + object
                + " v=" + ((Fragment)object).getView());
        mCurTransaction.detach((Fragment)object);
    }
```
由于在fragmentManager中对fragment进行了缓存，而且他在调用destroyItem方法时并没有销毁，而是通过事物进行detach，所以当改变adapter中的fragment数量时他会先findFragmentByTag(name)方法，而name参数是makefragmentName()而其中的参数getItemId(position)通常情况下是返回的Position，所以在改变fragment的数量的时候，调用notifydatasetchange方法的时候，由于之前已经缓存了对应的fragment，所以就会造成数据错乱，其实就是上边的两种方案，一是在初始化后重新设置数据，二是清除缓存，每次都重新创建。
### 对于FragmentStatePagerAdapter就很简单
由于其特性第只保留当前页面，其他页面不可见时就会销毁调用destroyitem，看下列源码：
```java
 @Override
    public Object instantiateItem(ViewGroup container, int position) {
        // If we already have this item instantiated, there is nothing
        // to do.  This can happen when we are restoring the entire pager
        // from its saved state, where the fragment manager has already
        // taken care of restoring the fragments we previously had instantiated.
        if (mFragments.size() > position) {
            Fragment f = mFragments.get(position);
            if (f != null) {
                return f;
            }
        }

        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }

        Fragment fragment = getItem(position);
        if (DEBUG) Log.v(TAG, "Adding item #" + position + ": f=" + fragment);
        if (mSavedState.size() > position) {
            Fragment.SavedState fss = mSavedState.get(position);
            if (fss != null) {
                fragment.setInitialSavedState(fss);
            }
        }
        while (mFragments.size() <= position) {
            mFragments.add(null);
        }
        fragment.setMenuVisibility(false);
        fragment.setUserVisibleHint(false);
        mFragments.set(position, fragment);
        mCurTransaction.add(container.getId(), fragment);

        return fragment;
    }

    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        Fragment fragment = (Fragment) object;

        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }
        if (DEBUG) Log.v(TAG, "Removing item #" + position + ": f=" + object
                + " v=" + ((Fragment)object).getView());
        while (mSavedState.size() <= position) {
            mSavedState.add(null);
        }
        mSavedState.set(position, fragment.isAdded()
                ? mFragmentManager.saveFragmentInstanceState(fragment) : null);
        mFragments.set(position, null);

        mCurTransaction.remove(fragment);
    }
```

从源码中可以看到他在调用destroyitem的时候会从fragmentManager中remove掉，每次都会重新生成一个fragment，所以我们只需要重写getitemPosition方法即可。

这个问题由于之前对Viewpager机制不太熟，所以导致的这个问题，一次来做一个记录吧！










参考文章：
- <https://www.cnblogs.com/dancefire/archive/2013/01/02/why-notifydatasetchanged-does-not-work.html>