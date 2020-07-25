---
title: SparseArray和ArrayMap原理解析
date: 2020-07-26 11:05
categories: 
- 源码学习
tags:
- android进阶
description: SparseArray和ArrayMap原理解析
---

## SparseArray和ArrayMap原理解析

https://blog.csdn.net/hq942845204/article/details/81293480

### SparseArray

该集合是由两个数组实现的，private int[] mKeys;       private Object[] mValues;

- 构造器

  ```jab
   /**
       * Creates a new SparseArray containing no mappings.
       */
      public SparseArray() {
      	//默认容量是 10
          this(10);
      }
      
       public SparseArray(int initialCapacity) {
          if (initialCapacity == 0) {
              mKeys = EmptyArray.INT;
              mValues = EmptyArray.OBJECT;
          } else {
              mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
              mKeys = new int[mValues.length];
          }
          mSize = 0;
      }
  ```

- ContainerHelpers.binarySearch通过二分查找进行查找key数组

  ```java
  static int binarySearch(int[] array, int size, int value) {
      int lo = 0;
      int hi = size - 1;
  
      while (lo <= hi) {
          final int mid = (lo + hi) >>> 1;
          final int midVal = array[mid];
  
          if (midVal < value) {
              lo = mid + 1;
          } else if (midVal > value) {
              hi = mid - 1;
          } else {
              return mid;  // value found
          }
      }
     //未找到对应的index，返回左侧变量值的取反，负数且再取反就是下一个要插入的索引位置
      return ~lo;  // value not present
  }
  ```

- put方法

  ```java
  public void put(int key, E value) {
    		//通过二分查找的方式进行查找，找到对应的索引或下一个插入的位置
          int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
  				//大于等于零说明找到位置或还未仿佛元素，直接设置即可
          if (i >= 0) {
              mValues[i] = value;
          } else {
            //这里取反对应没有找到位置时，二分查找返回结果取反
              i = ~i;
  						//如果是正好小于mSize，且value是无效状态
              if (i < mSize && mValues[i] == DELETED) {
                  mKeys[i] = key;
                  mValues[i] = value;
                  return;
              }
  						//如果集合需要进行回收是并且元素集合已经满了
              if (mGarbage && mSize >= mKeys.length) {
                  gc();
  
                  // Search again because indices may have changed.
                  i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
              }
  						//插入元素
              mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
              mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
              mSize++;
          }
      }
  ```



- GrowingArrayUtils.insert插入元素

  ```java
   public static boolean[] insert(boolean[] array, int currentSize, int index, boolean element) {
          assert currentSize <= array.length;
  				//直接插入
          if (currentSize + 1 <= array.length) {
              System.arraycopy(array, index, array, index + 1, currentSize - index);
              array[index] = element;
              return array;
          }
  				//扩容 currentSize <= 4 ? 8 : currentSize * 2;
          boolean[] newArray = new boolean[growSize(currentSize)];
     			//插入元素前半段复制
          System.arraycopy(array, 0, newArray, 0, index);
          newArray[index] = element;
     			//插入元素后半段复制
          System.arraycopy(array, index, newArray, index + 1, array.length - index);
          return newArray;
      }
  ```

  

- get方法

  ```java
   public E get(int key, E valueIfKeyNotFound) {
          int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
  
          if (i < 0 || mValues[i] == DELETED) {
            //默认值
              return valueIfKeyNotFound;
          } else {
              return (E) mValues[i];
          }
      }
  ```

- delete方法

  ```java
  public void delete(int key) {
          int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
  
          if (i >= 0) {
            //删除是直接把当前元素修改成DELETED值
              if (mValues[i] != DELETED) {
                  mValues[i] = DELETED;
                  mGarbage = true;
              }
          }
      }
  ```

- gc方法

```java
 private void gc() {
        // Log.e("SparseArray", "gc start with " + mSize);

        int n = mSize;
        int o = 0;
        int[] keys = mKeys;
        Object[] values = mValues;
			//循环遍历，删除DELETED状态的元素，并向前移动
        for (int i = 0; i < n; i++) {
            Object val = values[i];

            if (val != DELETED) {
                if (i != o) {
                    keys[o] = keys[i];
                    values[o] = val;
                    values[i] = null;
                }

                o++;
            }
        }

        mGarbage = false;
        mSize = o;

        // Log.e("SparseArray", "gc end with " + mSize);
    }
```





## ArrayMap

该集合和SparseArray类似，内部也是用两个数组实现的，分别是int[] mHashes;存储key对应的哈希值，Object[] mArray;成对存储key和value,key在偶数位，value在奇数位

- 构造器

  ```java
  public ArrayMap(int capacity, boolean identityHashCode) {
      mIdentityHashCode = identityHashCode;
  
      // If this is immutable, use the sentinal EMPTY_IMMUTABLE_INTS
      // instance instead of the usual EmptyArray.INT. The reference
      // is checked later to see if the array is allowed to grow.
      if (capacity < 0) {
          mHashes = EMPTY_IMMUTABLE_INTS;
          mArray = EmptyArray.OBJECT;
      } else if (capacity == 0) {
          mHashes = EmptyArray.INT;
          mArray = EmptyArray.OBJECT;
      } else {
          allocArrays(capacity);
      }
      mSize = 0;
  }
  ```

- put(K key, V value)

  使用过程中会有哈希冲突，解决方法是在通过二分法找到的索引位置前后进行查找

  key的位置=index<<2,     value的位置=index<<2+1

  ```java
  public V put(K key, V value) {
      final int osize = mSize;
      final int hash;
      int index;
      if (key == null) {
          hash = 0;
          index = indexOfNull();
      } else {
          hash = mIdentityHashCode ? System.identityHashCode(key) : key.hashCode();
          index = indexOf(key, hash);
      }
      if (index >= 0) {
          index = (index<<1) + 1;
          final V old = (V)mArray[index];
          mArray[index] = value;
          return old;
      }
  
      index = ~index;
    //未找到，需要扩容BASE_SIZE=4
      if (osize >= mHashes.length) {
          final int n = osize >= (BASE_SIZE*2) ? (osize+(osize>>1))
                  : (osize >= BASE_SIZE ? (BASE_SIZE*2) : BASE_SIZE);
  
          if (DEBUG) Log.d(TAG, "put: grow from " + mHashes.length + " to " + n);
  
          final int[] ohashes = mHashes;
          final Object[] oarray = mArray;
        //分配内存
          allocArrays(n);
  
          if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
              throw new ConcurrentModificationException();
          }
  				//将原来的元素分别复制到新集合中
          if (mHashes.length > 0) {
              if (DEBUG) Log.d(TAG, "put: copy 0-" + osize + " to 0");
              System.arraycopy(ohashes, 0, mHashes, 0, ohashes.length);
              System.arraycopy(oarray, 0, mArray, 0, oarray.length);
          }
  				//释放旧集合空间
          freeArrays(ohashes, oarray, osize);
      }
  //如果当前元素在中间，需要移动当前索引后边的元素向后移动一个位置
      if (index < osize) {
          if (DEBUG) Log.d(TAG, "put: move " + index + "-" + (osize-index)
                  + " to " + (index+1));
          System.arraycopy(mHashes, index, mHashes, index + 1, osize - index);
          System.arraycopy(mArray, index << 1, mArray, (index + 1) << 1, (mSize - index) << 1);
      }
  
      if (CONCURRENT_MODIFICATION_EXCEPTIONS) {
          if (osize != mSize || index >= mHashes.length) {
              throw new ConcurrentModificationException();
          }
      }
    //给当前元素赋值
      mHashes[index] = hash;
      mArray[index<<1] = key;
      mArray[(index<<1)+1] = value;
      mSize++;
      return null;
  }
  ```

- indexOf(Object key, int hash)

  ```java
  int indexOf(Object key, int hash) {
      final int N = mSize;
  
      // Important fast case: if nothing is in here, nothing to look for.
      if (N == 0) {
          return ~0;
      }
  
      int index = binarySearchHashes(mHashes, N, hash);
  
      // If the hash code wasn't found, then we have no entry for this key.
      if (index < 0) {
          return index;
      }
  
      // If the key at the returned index matches, that's what we want.
      if (key.equals(mArray[index<<1])) {
          return index;
      }
  
      // Search for a matching key after the index.
    //存在哈希冲突，向后查找
      int end;
      for (end = index + 1; end < N && mHashes[end] == hash; end++) {
          if (key.equals(mArray[end << 1])) return end;
      }
  
      // Search for a matching key before the index.
     //存在哈希冲突，向前查找
      for (int i = index - 1; i >= 0 && mHashes[i] == hash; i--) {
          if (key.equals(mArray[i << 1])) return i;
      }
  
      // Key not found -- return negative value indicating where a
      // new entry for this key should go.  We use the end of the
      // hash chain to reduce the number of array entries that will
      // need to be copied when inserting.
      return ~end;
  }
  ```







































































