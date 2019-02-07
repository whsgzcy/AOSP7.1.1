# List

* **一、Activity**
* **1.1 LayoutInflater.Factory2**
* **1.2 ContextThemeWrapper**
* **1.3 ComponentCallbacks2**

* **二、BroadcastReceiver**

## 一、Activity

```
/core/java/android/app/Activity.java
```

有过Android APP开发经验的人看到代码上的注释其实是很容易理解的，就可以理解为，专注于用户交互的一个single，它可以被用作于xxx，等；

```
/**
 * An activity is a single, focused thing that the user can do.  Almost all
 * activities interact with the user, so the Activity class takes care of
 * creating a window for you in which you can place your UI with
 * {@link #setContentView}.  While activities are often presented to the user
 * as full-screen windows, they can also be used in other ways: as floating
 * windows (via a theme with {@link android.R.attr#windowIsFloating} set)
 * or embedded inside of another activity (using {@link ActivityGroup}).
 *
 ...
```

如果只看开头的这段代码来说，他就是实现这些接口的回调，事件、window、组件等等，代码量很多，在此只做两点分析：

* LayoutInflater.Factory2
* ContextThemeWrapper
* ComponentCallbacks2

```
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback, WindowControllerCallback
```

### 1.1 LayoutInflater.Factory2

其实看到LayoutInflater，和我们平时在写view时，xml转换为view其实是一样的，它的本质还是基于一个反射机制；

实例化xml文件对应其view

```
/core/java/android/view/LayoutInflater.java
```

```
/**
 * Instantiates a layout XML file into its corresponding {@link android.view.View}
 * objects. It is never used directly. Instead, use
 * {@link android.app.Activity#getLayoutInflater()} or
 * {@link Context#getSystemService} to retrieve a standard LayoutInflater instance
 * that is already hooked up to the current context and correctly configured
 * for the device you are running on.  For example:
 *
 * <pre>LayoutInflater inflater = (LayoutInflater)context.getSystemService
 *      (Context.LAYOUT_INFLATER_SERVICE);</pre>
 *
 * <p>
 * To create a new LayoutInflater with an additional {@link Factory} for your
 * own views, you can use {@link #cloneInContext} to clone an existing
 * ViewFactory, and then call {@link #setFactory} on it to include your
 * Factory.
 *
 * <p>
 * For performance reasons, view inflation relies heavily on pre-processing of
 * XML files that is done at build time. Therefore, it is not currently possible
 * to use LayoutInflater with an XmlPullParser over a plain XML file at runtime;
 * it only works with an XmlPullParser returned from a compiled resource
 * (R.<em>something</em> file.)
 *
 * @see Context#getSystemService
 */
```

那么Factory2是什么

在系统填充view前会回调该接口，你可以去自定义布局的填充（有点类似于拦截器）

使用LayoutInflater Factory的这一特性可以做很多事：

* 提高view构建的效率

当我们使用自定义view时，需要在xml中使用完整类名，系统实际就是根据完整类名进行反射构建。我们可以自己new出view避免系统反射调用，提高效率；

* 替换默认view实现，改变或添加属性

**替换系统控件**

**一键换肤解决方案**

```
...
  public interface Factory {
        /**
         * Hook you can supply that is called when inflating from a LayoutInflater.
         * You can use this to customize the tag names available in your XML
         * layout files.
         * 
         * <p>
         * Note that it is good practice to prefix these custom names with your
         * package (i.e., com.coolcompany.apps) to avoid conflicts with system
         * names.
         * 
         * @param name Tag name to be inflated.
         * @param context The context the view is being created in.
         * @param attrs Inflation attributes as specified in XML file.
         * 
         * @return View Newly created view. Return null for the default
         *         behavior.
         */
        public View onCreateView(String name, Context context, AttributeSet attrs);
    }

    public interface Factory2 extends Factory {
        /**
         * Version of {@link #onCreateView(String, Context, AttributeSet)}
         * that also supplies the parent that the view created view will be
         * placed in.
         *
         * @param parent The parent that the created view will be placed
         * in; <em>note that this may be null</em>.
         * @param name Tag name to be inflated.
         * @param context The context the view is being created in.
         * @param attrs Inflation attributes as specified in XML file.
         *
         * @return View Newly created view. Return null for the default
         *         behavior.
         */
        public View onCreateView(View parent, String name, Context context, AttributeSet attrs);
    }
    
    ...
    
    /**
     * Create a new LayoutInflater instance associated with a particular Context.
     * Applications will almost always want to use
     * {@link Context#getSystemService Context.getSystemService()} to retrieve
     * the standard {@link Context#LAYOUT_INFLATER_SERVICE Context.INFLATER_SERVICE}.
     * 
     * @param context The Context in which this LayoutInflater will create its
     * Views; most importantly, this supplies the theme from which the default
     * values for their attributes are retrieved.
     */
    protected LayoutInflater(Context context) {
        mContext = context;
    }

    /**
     * Create a new LayoutInflater instance that is a copy of an existing
     * LayoutInflater, optionally with its Context changed.  For use in
     * implementing {@link #cloneInContext}.
     * 
     * @param original The original LayoutInflater to copy.
     * @param newContext The new Context to use.
     */
    protected LayoutInflater(LayoutInflater original, Context newContext) {
        mContext = newContext;
        mFactory = original.mFactory;
        mFactory2 = original.mFactory2;
        mPrivateFactory = original.mPrivateFactory;
        setFilter(original.mFilter);
    }
    
    /**
     * Obtains the LayoutInflater from the given context.
     */
    public static LayoutInflater from(Context context) {
        LayoutInflater LayoutInflater =
                (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        if (LayoutInflater == null) {
            throw new AssertionError("LayoutInflater not found.");
        }
        return LayoutInflater;
    }

    /**
     * Create a copy of the existing LayoutInflater object, with the copy
     * pointing to a different Context than the original.  This is used by
     * {@link ContextThemeWrapper} to create a new LayoutInflater to go along
     * with the new Context theme.
     * 
     * @param newContext The new Context to associate with the new LayoutInflater.
     * May be the same as the original Context if desired.
     * 
     * @return Returns a brand spanking new LayoutInflater object associated with
     * the given Context.
     */
    public abstract LayoutInflater cloneInContext(Context newContext);
    
    /**
     * Return the context we are running in, for access to resources, class
     * loader, etc.
     */
    public Context getContext() {
        return mContext;
    }

    /**
     * Return the current {@link Factory} (or null). This is called on each element
     * name. If the factory returns a View, add that to the hierarchy. If it
     * returns null, proceed to call onCreateView(name).
     */
    public final Factory getFactory() {
        return mFactory;
    }

    /**
     * Return the current {@link Factory2}.  Returns null if no factory is set
     * or the set factory does not implement the {@link Factory2} interface.
     * This is called on each element
     * name. If the factory returns a View, add that to the hierarchy. If it
     * returns null, proceed to call onCreateView(name).
     */
    public final Factory2 getFactory2() {
        return mFactory2;
    }

    /**
     * Attach a custom Factory interface for creating views while using
     * this LayoutInflater.  This must not be null, and can only be set once;
     * after setting, you can not change the factory.  This is
     * called on each element name as the xml is parsed. If the factory returns
     * a View, that is added to the hierarchy. If it returns null, the next
     * factory default {@link #onCreateView} method is called.
     * 
     * <p>If you have an existing
     * LayoutInflater and want to add your own factory to it, use
     * {@link #cloneInContext} to clone the existing instance and then you
     * can use this function (once) on the returned new instance.  This will
     * merge your own factory with whatever factory the original instance is
     * using.
     */
    public void setFactory(Factory factory) {
        if (mFactorySet) {
            throw new IllegalStateException("A factory has already been set on this LayoutInflater");
        }
        if (factory == null) {
            throw new NullPointerException("Given factory can not be null");
        }
        mFactorySet = true;
        if (mFactory == null) {
            mFactory = factory;
        } else {
            mFactory = new FactoryMerger(factory, null, mFactory, mFactory2);
        }
    }

    /**
     * Like {@link #setFactory}, but allows you to set a {@link Factory2}
     * interface.
     */
    public void setFactory2(Factory2 factory) {
        if (mFactorySet) {
            throw new IllegalStateException("A factory has already been set on this LayoutInflater");
        }
        if (factory == null) {
            throw new NullPointerException("Given factory can not be null");
        }
        mFactorySet = true;
        if (mFactory == null) {
            mFactory = mFactory2 = factory;
        } else {
            mFactory = mFactory2 = new FactoryMerger(factory, factory, mFactory, mFactory2);
        }
    }

    /**
     * @hide for use by framework
     */
    public void setPrivateFactory(Factory2 factory) {
        if (mPrivateFactory == null) {
            mPrivateFactory = factory;
        } else {
            mPrivateFactory = new FactoryMerger(factory, factory, mPrivateFactory, mPrivateFactory);
        }
    }
    ...
```

### 1.2 ContextThemeWrapper

从注解上来看，是允许或者修改context所包含的主题，从用法上来看，他是从style中拿属性填充view，也就是我们把style封装成xml的形式给外部使用，但是其解释为“... allows you to modify or replace the theme of the wrapped context.” 这个context所包含的，那么这个context是什么呢？

```
/core/java/android/view/ContextThemeWrapper.java

/**
 * A context wrapper that allows you to modify or replace the theme of the
 * wrapped context.
 */
public class ContextThemeWrapper extends ContextWrapper {
    private int mThemeResource;
    private Resources.Theme mTheme;
    private LayoutInflater mInflater;
    private Configuration mOverrideConfiguration;
    private Resources mResources;
    ...
```

阅读ContextWrapper.java的代码，有很多context常见的方法，比如“sendBroadcast ...”等，从代码注释中也可以看出，他是一个简单的代理，可以对其子类进行修改；

**问题1 这里引入一个问题，context.sendBroadCast(xxx)，发送就会有遍历，遍历就可能会有匹配，接收，那么他是怎么发送，怎么接收？**

```
/core/java/android/content/ContextWrapper.java

/**
 * Proxying implementation of Context that simply delegates all of its calls to
 * another Context.  Can be subclassed to modify behavior without changing
 * the original Context.
 */
public class ContextWrapper extends Context {
    Context mBase;
    ...
```
有关应用程序环境的全局信息的接口。 这是一个抽象类，其实现由Android系统提供。 它允许访问特定于应用程序的资源和类，以及对应用程序级操作的上调，例如启动活动，广播和接收意图等；

在Activity中可以调用this.sendBroadCastXXX,所以Activity的本质是一个context，也就是上述“ ... the wrapped context.” 这个context所包含的，context是一个App的主线！

```
/core/java/android/content/Context.java

/**
 * Interface to global information about an application environment.  This is
 * an abstract class whose implementation is provided by
 * the Android system.  It
 * allows access to application-specific resources and classes, as well as
 * up-calls for application-level operations such as launching activities,
 * broadcasting and receiving intents, etc.
 */
public abstract class Context {
    /**
     * File creation mode: the default mode, where the created file can only
     * be accessed by the calling application (or all applications sharing the
     * same user ID).
     */
    public static final int MODE_PRIVATE = 0x0000;
    ...
```

### 1.3 ComponentCallbacks2

[点击](https://blog.csdn.net/time_hunter/article/details/53107191)

**问题2 我们都知道Android是Linux裁剪之后的产物，例如：1、Exception，他的是实质就是判断指针的类型，若指针为null则jni返回给java的即是null，outOfArray等为.size的判断；2、Log，我们通过adb都可以从buffer中读到日志，他的实质也是Linux所特有，只是通过jni返回上去，那么我们在使用360等内存优化软件时，从application至少可以读取到内存和cpu等，那么他是不是也是Linux中top 通过jni返回上去的呢？在哪进行的调用？**

```
core/java/android/content/ComponentCallbacks2.java

/**
     * Level for {@link #onTrimMemory(int)}: the process is nearing the end
     * of the background LRU list, and if more memory isn't found soon it will
     * be killed.
     */
    static final int TRIM_MEMORY_COMPLETE = 80;
    
    /**
     * Level for {@link #onTrimMemory(int)}: the process is around the middle
     * of the background LRU list; freeing memory can help the system keep
     * other processes running later in the list for better overall performance.
     */
    static final int TRIM_MEMORY_MODERATE = 60;
    
    /**
     * Level for {@link #onTrimMemory(int)}: the process has gone on to the
     * LRU list.  This is a good opportunity to clean up resources that can
     * efficiently and quickly be re-built if the user returns to the app.
     */
    static final int TRIM_MEMORY_BACKGROUND = 40;
    
    /**
     * Level for {@link #onTrimMemory(int)}: the process had been showing
     * a user interface, and is no longer doing so.  Large allocations with
     * the UI should be released at this point to allow memory to be better
     * managed.
     */
    static final int TRIM_MEMORY_UI_HIDDEN = 20;

    /**
     * Level for {@link #onTrimMemory(int)}: the process is not an expendable
     * background process, but the device is running extremely low on memory
     * and is about to not be able to keep any background processes running.
     * Your running process should free up as many non-critical resources as it
     * can to allow that memory to be used elsewhere.  The next thing that
     * will happen after this is {@link #onLowMemory()} called to report that
     * nothing at all can be kept in the background, a situation that can start
     * to notably impact the user.
     */
    static final int TRIM_MEMORY_RUNNING_CRITICAL = 15;

    /**
     * Level for {@link #onTrimMemory(int)}: the process is not an expendable
     * background process, but the device is running low on memory.
     * Your running process should free up unneeded resources to allow that
     * memory to be used elsewhere.
     */
    static final int TRIM_MEMORY_RUNNING_LOW = 10;


    /**
     * Level for {@link #onTrimMemory(int)}: the process is not an expendable
     * background process, but the device is running moderately low on memory.
     * Your running process may want to release some unneeded resources for
     * use elsewhere.
     */
    static final int TRIM_MEMORY_RUNNING_MODERATE = 5;

    /**
     * Called when the operating system has determined that it is a good
     * time for a process to trim unneeded memory from its process.  This will
     * happen for example when it goes in the background and there is not enough
     * memory to keep as many background processes running as desired.  You
     * should never compare to exact values of the level, since new intermediate
     * values may be added -- you will typically want to compare if the value
     * is greater or equal to a level you are interested in.
     *
     * <p>To retrieve the processes current trim level at any point, you can
     * use {@link android.app.ActivityManager#getMyMemoryState
     * ActivityManager.getMyMemoryState(RunningAppProcessInfo)}.
     *
     * @param level The context of the trim, giving a hint of the amount of
     * trimming the application may like to perform.  May be
     * {@link #TRIM_MEMORY_COMPLETE}, {@link #TRIM_MEMORY_MODERATE},
     * {@link #TRIM_MEMORY_BACKGROUND}, {@link #TRIM_MEMORY_UI_HIDDEN},
     * {@link #TRIM_MEMORY_RUNNING_CRITICAL}, {@link #TRIM_MEMORY_RUNNING_LOW},
     * or {@link #TRIM_MEMORY_RUNNING_MODERATE}.
     */
    void onTrimMemory(int level);
```

## 二、BroadcastReceiver

广播，其实也是一种消息传递机制，Intent其实本质是一个序列化的对象，消息的载体:

```
/core/java/android/content/Intent.java
```

他可以在进程内通信，也可以在进程间通信，千万不能过于频繁，能不用广播就不用广播；

现在问题来了，





问题1：

问题2：

startService和bindService


参考：

https://blog.csdn.net/u013085697/article/details/53898879

https://blog.csdn.net/time_hunter/article/details/53107191