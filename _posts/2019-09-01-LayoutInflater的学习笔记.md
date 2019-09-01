---
title: LayoutInflater的学习笔记
date: 2019-09-01 11:05
categories: 
- 源码分析
tags:
- android进阶
description: LayoutInflater的学习笔记
---

# LayoutInflater加载布局过程解析

## 获取LayoutInflater实例

> **通过LayoutInflater.from(Context context)来获取**

```java
public static LayoutInflater from(Context context) {
        LayoutInflater LayoutInflater =
                (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        if (LayoutInflater == null) {
            throw new AssertionError("LayoutInflater not found.");
        }
        return LayoutInflater;
    }
```

>  **context.getSystemService(Context.LAYOUT_INFLATER_SERVICE)**

```java
 LayoutInflater LayoutInflater =
                (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
```



> **如果是在Activity中，可以通过Activity的getLayoutInflater方法获取，**

```java
public LayoutInflater getLayoutInflater() {        
  return getWindow().getLayoutInflater();    
}
```

```java
public PhoneWindow(Context context) {
        super(context);
        mLayoutInflater = LayoutInflater.from(context);
    }

```

## inflate填充布局

> 第一种

```java

public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    return inflate(resource, root, root != null);
}
```

> 第二种

```java
 public View inflate(XmlPullParser parser, @Nullable ViewGroup root) {
        return inflate(parser, root, root != null);
    }
```

> 第三种

```java
 public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }

        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
```

> 第四种

```java
 public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;

            try {
                // Look for the root node.
                int type;
              //该while循环的作用应该是寻找  整个布局文件的根节点
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }

                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }
								//寻找根节点名字
                final String name = parser.getName();

                if (DEBUG) {
                    System.out.println("**************************");
                    System.out.println("Creating root view: "
                            + name);
                    System.out.println("**************************");
                }
										/*如果根节点是merge标签
                      *但是父View，即root为null,或不添加到root中，即attachToRoot为false    
                      *会报错  
                      */  
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }
										//递归生成子节点
                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                  //通过createViewFromTag生成根节点View 
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                          /*
                               * 如果不需要将根节点View添加到父View中，
                               * 则对根节点View自身设置LayoutParams    
                               */ 
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }

                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);

                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                      /*
                           * 如果父View不为null,并且根节点需要添加到父View中  
                           * 则调用root的addView方法添加根节点的View    
                           */
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } catch (XmlPullParserException e) {
                ...
            } finally {
            ...
            }

            return result;
        }
    }
```

> **createViewFromTag创建View**

```java
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {
    if (name.equals("view")) {
        name = attrs.getAttributeValue(null, "class");
    }

    // Apply a theme wrapper, if allowed and one is specified.
    if (!ignoreThemeAttr) {
        final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
        final int themeResId = ta.getResourceId(0, 0);
        if (themeResId != 0) {
            context = new ContextThemeWrapper(context, themeResId);
        }
        ta.recycle();
    }

    if (name.equals(TAG_1995)) {
        // Let's party like it's 1995!
        return new BlinkLayout(context, attrs);
    }

    try {
        View view;
      /*
               * Factory2和Factory是两个接口，
               * 如果我们自己在代码中设置了mFactory2或者mFactory， 
               * 在createViewFromTag()方法中创建View的时候就会调用mFactory2或者mFactory                                                                                    
               * 的onCrateView()方法，通过我们自己实现的onCreateView()来创建View
               * 在侵入式的插件换肤框架的设计上，就利用了mFactory2这个对象，
               * 实现了onCreateView，用来hook View的创建过程，标记需要被换肤的View  
               */ 
        if (mFactory2 != null) {
            view = mFactory2.onCreateView(parent, name, context, attrs);
        } else if (mFactory != null) {
            view = mFactory.onCreateView(name, context, attrs);
        } else {
            view = null;
        }

        if (view == null && mPrivateFactory != null) {
            view = mPrivateFactory.onCreateView(parent, name, context, attrs);
        }
/*
               * 如果没有设置mFactory2和mFactory，或者最后return null的话，  
               * 就会执行以下代码，最终通过反射的方式构建View 
               */
        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
                if (-1 == name.indexOf('.')) {
                    view = onCreateView(parent, name, attrs);
                } else {
                    view = createView(name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }

        return view;
    } catch (InflateException e) {
        throw e;

    } catch (ClassNotFoundException e) {
        final InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class " + name, e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;

    } catch (Exception e) {
        final InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class " + name, e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;
    }
}
```

看一下AppCompatActivity的部分代码：

```java
    private AppCompatDelegate mDelegate;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        final AppCompatDelegate delegate = getDelegate();
        //注意这个方法
        delegate.installViewFactory();
        delegate.onCreate(savedInstanceState);
        if (delegate.applyDayNight() && mThemeId != 0) {
            // If DayNight has been applied, we need to re-apply the theme for
            // the changes to take effect. On API 23+, we should bypass
            // setTheme(), which will no-op if the theme ID is identical to the
            // current theme ID.
            if (Build.VERSION.SDK_INT >= 23) {
                onApplyThemeResource(getTheme(), mThemeId, false);
            } else {
                setTheme(mThemeId);
            }
        }
        super.onCreate(savedInstanceState);
    }

    public AppCompatDelegate getDelegate() {
        if (mDelegate == null) {
            mDelegate = AppCompatDelegate.create(this, this);
        }
        return mDelegate;
    }
```

注意到有个`AppCompatDelegate.installViewFactory()`方法，该方法的实现在

AppCompatDelegateIml中

```java
@Override
public void installViewFactory() {
    LayoutInflater layoutInflater = LayoutInflater.from(mContext);
    if (layoutInflater.getFactory() == null) {
      //AppCompatDelegateImpl本身实现了LayoutInflater.Factory2接口，
            //调用LayoutInflater.setFactory2方法来设置自身
        LayoutInflaterCompat.setFactory2(layoutInflater, this);
    } else {
        if (!(layoutInflater.getFactory2() instanceof AppCompatDelegateImpl)) {
            Log.i(TAG, "The Activity's LayoutInflater already has a Factory installed"
                    + " so we can not install AppCompat's");
        }
    }
}
```

那么我们看下实现的onCreateView方法：

```java
/**
 * From {@link LayoutInflater.Factory2}.
 */
@Override
public final View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
    return createView(parent, name, context, attrs);
}

/**
 * From {@link LayoutInflater.Factory2}.
 */
@Override
public View onCreateView(String name, Context context, AttributeSet attrs) {
    return onCreateView(null, name, context, attrs);
}
```

最终会调用AppCompatViewInflater.createView

```java
@Override
public View createView(View parent, final String name, @NonNull Context context,
        @NonNull AttributeSet attrs) {
  ...
     return mAppCompatViewInflater.createView(parent, name, context, attrs, inheritContext,
                IS_PRE_LOLLIPOP, /* Only read android:theme pre-L (L+ handles this anyway) */
                true, /* Read read app:theme as a fallback at all times for legacy reasons */
                VectorEnabledTintResources.shouldBeUsed() /* Only tint wrap the context if enabled */
        );
}
```

AppCompatViewInflater.createView()

```java
final View createView(View parent, final String name, @NonNull Context context,
        @NonNull AttributeSet attrs, boolean inheritContext,
        boolean readAndroidTheme, boolean readAppTheme, boolean wrapContext) {
    final Context originalContext = context;

   ...

    View view = null;

    // We need to 'inject' our tint aware Views in place of the standard framework versions
    switch (name) {
        case "TextView":
            view = createTextView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "ImageView":
            view = createImageView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "Button":
            view = createButton(context, attrs);
            verifyNotNull(view, name);
            break;
        case "EditText":
            view = createEditText(context, attrs);
            verifyNotNull(view, name);
            break;
        case "Spinner":
            view = createSpinner(context, attrs);
            verifyNotNull(view, name);
            break;
        case "ImageButton":
            view = createImageButton(context, attrs);
            verifyNotNull(view, name);
            break;
        case "CheckBox":
            view = createCheckBox(context, attrs);
            verifyNotNull(view, name);
            break;
        case "RadioButton":
            view = createRadioButton(context, attrs);
            verifyNotNull(view, name);
            break;
        case "CheckedTextView":
            view = createCheckedTextView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "AutoCompleteTextView":
            view = createAutoCompleteTextView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "MultiAutoCompleteTextView":
            view = createMultiAutoCompleteTextView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "RatingBar":
            view = createRatingBar(context, attrs);
            verifyNotNull(view, name);
            break;
        case "SeekBar":
            view = createSeekBar(context, attrs);
            verifyNotNull(view, name);
            break;
        default:
            // The fallback that allows extending class to take over view inflation
            // for other tags. Note that we don't check that the result is not-null.
            // That allows the custom inflater path to fall back on the default one
            // later in this method.
            view = createView(context, name, attrs);
    }

    if (view == null && originalContext != context) {
        // If the original context does not equal our themed context, then we need to manually
        // inflate it using the name so that android:theme takes effect.
        view = createViewFromTag(context, name, attrs);
    }

    if (view != null) {
        // If we have created a view, check its android:onClick
        checkOnClickListener(view, attrs);
    }

    return view;
}
```

看到这里有没有感觉很熟悉？该方法会将Android通用的组件替换成AppCompatXXX这些组件，虽然我们在XML文件中写的是ImageView，但实际得到的是AppCompatImageView。

```java
 @NonNull
    protected AppCompatImageView createImageView(Context context, AttributeSet attrs) {
        return new AppCompatImageView(context, attrs);
    }
```

上面creatView的源码最后有一句：

```java
if (view != null) {
        // If we have created a view, check its android:onClick
        checkOnClickListener(view, attrs);
    }
```

checkOnClickListener用来检测View是否有`android:onClick`属性，若有，则为其添加点击事件，方法实现：

```java
/**
 * android:onClick doesn't handle views with a ContextWrapper context. This method
 * backports new framework functionality to traverse the Context wrappers to find a
 * suitable target.
 */
private void checkOnClickListener(View view, AttributeSet attrs) {
    final Context context = view.getContext();

    if (!(context instanceof ContextWrapper) ||
            (Build.VERSION.SDK_INT >= 15 && !ViewCompat.hasOnClickListeners(view))) {
        // Skip our compat functionality if: the Context isn't a ContextWrapper, or
        // the view doesn't have an OnClickListener (we can only rely on this on API 15+ so
        // always use our compat code on older devices)
        return;
    }

    final TypedArray a = context.obtainStyledAttributes(attrs, sOnClickAttrs);
    final String handlerName = a.getString(0);
    if (handlerName != null) {
        view.setOnClickListener(new DeclaredOnClickListener(view, handlerName));
    }
    a.recycle();
}
```

可以看到，这里给View设置了`DeclaredOnClickListener`，该类内部通过反射调用了我们设置的`android:onClick="XXX"`中的“XXX”方法。同样的，View类内部处理该属性也是同样的方式。也就是说，使用`android:onClick`属性，会创建一个中间类，然后反射调用我们设置的回调方法，效率肯定没有在java代码设置一个onClickListener高。





其他详解参考：[https://www.jianshu.com/p/6a193b61a833](https://www.jianshu.com/p/6a193b61a833)