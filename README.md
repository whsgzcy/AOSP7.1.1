# List

* **一、Activity**
* **1.1 LayoutInflater.Factory2**
* **1.2 ContextThemeWrapper**
* **1.3 ComponentCallbacks2**

* **二、BroadcastReceiver**

* **三、Binder**
* **3.1 什么是IPC**
* **3.2 Android中的IPC Binder**
* **3.3 获取服务流程**

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

对于广播，他到底是怎么把数据传过去的？**问题1答案**

![MacDown logo](https://github.com/whsgzcy/AOSP7.1.1/blob/master/images/broadcast.jpg)

XXXActivity为注册广播和发送广播；

```
core/java/android/app/LoadedApk.java

...
  private final ArrayMap<Context, ArrayMap<BroadcastReceiver, ReceiverDispatcher>> mReceivers
        = new ArrayMap<Context, ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>>();
    private final ArrayMap<Context, ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>> mUnregisteredReceivers
        = new ArrayMap<Context, ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>>();
    private final ArrayMap<Context, ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>> mServices
        = new ArrayMap<Context, ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>>();
    private final ArrayMap<Context, ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>> mUnboundServices
        = new ArrayMap<Context, ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>>();
        ...

public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
            Context context, Handler handler,
            Instrumentation instrumentation, boolean registered) {
        synchronized (mReceivers) {
            LoadedApk.ReceiverDispatcher rd = null;
            ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = null;
            if (registered) {
                map = mReceivers.get(context);
                if (map != null) {
                    rd = map.get(r);
                }
            }
            if (rd == null) {
                rd = new ReceiverDispatcher(r, context, handler,
                        instrumentation, registered);
                if (registered) {
                    if (map == null) {
                        map = new ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
                        mReceivers.put(context, map);
                    }
                    map.put(r, rd);
                }
            } else {
                rd.validate(context, handler);
            }
            rd.mForgotten = false;
            return rd.getIIntentReceiver();
        }
    }
    ...
    public void performReceive(Intent intent, int resultCode, String data,
                Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
            final Args args = new Args(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
            if (intent == null) {
                Log.wtf(TAG, "Null intent received");
            } else {
                if (ActivityThread.DEBUG_BROADCAST) {
                    int seq = intent.getIntExtra("seq", -1);
                    Slog.i(ActivityThread.TAG, "Enqueueing broadcast " + intent.getAction()
                            + " seq=" + seq + " to " + mReceiver);
                }
            }
            if (intent == null || !mActivityThread.post(args)) {
                if (mRegistered && ordered) {
                    IActivityManager mgr = ActivityManagerNative.getDefault();
                    if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                            "Finishing sync broadcast to " + mReceiver);
                    args.sendFinished(mgr);
                }
            }
        }

    }
    ...
```

```
core/java/android/app/ActivityThread.java
...
/**
 * This manages the execution of the main thread in an
 * application process, scheduling and executing activities,
 * broadcasts, and other operations on it as the activity
 * manager requests.
 *
 * {@hide}
 */
 ...
```
```
...
@Override
    public void sendBroadcast(Intent intent) {
        mBase.sendBroadcast(intent);
    }
    ...
```

```
services/core/java/com/android/server/am/ActivityManagerService.java

...
/**
     * Keeps track of all IIntentReceivers that have been registered for broadcasts.
     * Hash keys are the receiver IBinder, hash value is a ReceiverList.
     */
    final HashMap<IBinder, ReceiverList> mRegisteredReceivers = new HashMap<>();
    
    /**
     * Resolver for broadcast intents to registered receivers.
     * Holds BroadcastFilter (subclass of IntentFilter).
     */
    final IntentResolver<BroadcastFilter, BroadcastFilter> mReceiverResolver
            = new IntentResolver<BroadcastFilter, BroadcastFilter>() {
        @Override
        protected boolean allowFilterResult(
                BroadcastFilter filter, List<BroadcastFilter> dest) {
            IBinder target = filter.receiverList.receiver.asBinder();
            for (int i = dest.size() - 1; i >= 0; i--) {
                if (dest.get(i).receiverList.receiver.asBinder() == target) {
                    return false;
                }
            }
            return true;
        }
        ...
        
          public Intent registerReceiver(IApplicationThread caller, String callerPackage,
            IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {
        enforceNotIsolatedCaller("registerReceiver");
        ArrayList<Intent> stickyIntents = null;
        ProcessRecord callerApp = null;
        int callingUid;
        int callingPid;
        synchronized(this) {
            if (caller != null) {
                callerApp = getRecordForAppLocked(caller);
                if (callerApp == null) {
                    throw new SecurityException(
                            "Unable to find app for caller " + caller
                            + " (pid=" + Binder.getCallingPid()
                            + ") when registering receiver " + receiver);
                }
                if (callerApp.info.uid != Process.SYSTEM_UID &&
                        !callerApp.pkgList.containsKey(callerPackage) &&
                        !"android".equals(callerPackage)) {
                    throw new SecurityException("Given caller package " + callerPackage
                            + " is not running in process " + callerApp);
                }
                callingUid = callerApp.info.uid;
                callingPid = callerApp.pid;
            } else {
                callerPackage = null;
                callingUid = Binder.getCallingUid();
                callingPid = Binder.getCallingPid();
            }

            userId = mUserController.handleIncomingUser(callingPid, callingUid, userId, true,
                    ALLOW_FULL_ONLY, "registerReceiver", callerPackage);

            Iterator<String> actions = filter.actionsIterator();
            if (actions == null) {
                ArrayList<String> noAction = new ArrayList<String>(1);
                noAction.add(null);
                actions = noAction.iterator();
            }

            // Collect stickies of users
            int[] userIds = { UserHandle.USER_ALL, UserHandle.getUserId(callingUid) };
            while (actions.hasNext()) {
                String action = actions.next();
                for (int id : userIds) {
                    ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(id);
                    if (stickies != null) {
                        ArrayList<Intent> intents = stickies.get(action);
                        if (intents != null) {
                            if (stickyIntents == null) {
                                stickyIntents = new ArrayList<Intent>();
                            }
                            stickyIntents.addAll(intents);
                        }
                    }
                }
            }
        }

        ArrayList<Intent> allSticky = null;
        if (stickyIntents != null) {
            final ContentResolver resolver = mContext.getContentResolver();
            // Look for any matching sticky broadcasts...
            for (int i = 0, N = stickyIntents.size(); i < N; i++) {
                Intent intent = stickyIntents.get(i);
                // If intent has scheme "content", it will need to acccess
                // provider that needs to lock mProviderMap in ActivityThread
                // and also it may need to wait application response, so we
                // cannot lock ActivityManagerService here.
                if (filter.match(resolver, intent, true, TAG) >= 0) {
                    if (allSticky == null) {
                        allSticky = new ArrayList<Intent>();
                    }
                    allSticky.add(intent);
                }
            }
        }

        // The first sticky in the list is returned directly back to the client.
        Intent sticky = allSticky != null ? allSticky.get(0) : null;
        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Register receiver " + filter + ": " + sticky);
        if (receiver == null) {
            return sticky;
        }

        synchronized (this) {
            if (callerApp != null && (callerApp.thread == null
                    || callerApp.thread.asBinder() != caller.asBinder())) {
                // Original caller already died
                return null;
            }
            ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
            if (rl == null) {
                rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                        userId, receiver);
                if (rl.app != null) {
                    rl.app.receivers.add(rl);
                } else {
                    try {
                        receiver.asBinder().linkToDeath(rl, 0);
                    } catch (RemoteException e) {
                        return sticky;
                    }
                    rl.linkedToDeath = true;
                }
                mRegisteredReceivers.put(receiver.asBinder(), rl);
            } else if (rl.uid != callingUid) {
                throw new IllegalArgumentException(
                        "Receiver requested to register for uid " + callingUid
                        + " was previously registered for uid " + rl.uid);
            } else if (rl.pid != callingPid) {
                throw new IllegalArgumentException(
                        "Receiver requested to register for pid " + callingPid
                        + " was previously registered for pid " + rl.pid);
            } else if (rl.userId != userId) {
                throw new IllegalArgumentException(
                        "Receiver requested to register for user " + userId
                        + " was previously registered for user " + rl.userId);
            }
            BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                    permission, callingUid, userId);
            rl.add(bf);
            if (!bf.debugCheck()) {
                Slog.w(TAG, "==> For Dynamic broadcast");
            }
            mReceiverResolver.addFilter(bf);

            // Enqueue broadcasts for all existing stickies that match
            // this filter.
            if (allSticky != null) {
                ArrayList receivers = new ArrayList();
                receivers.add(bf);

                final int stickyCount = allSticky.size();
                for (int i = 0; i < stickyCount; i++) {
                    Intent intent = allSticky.get(i);
                    BroadcastQueue queue = broadcastQueueForIntent(intent);
                    BroadcastRecord r = new BroadcastRecord(queue, intent, null,
                            null, -1, -1, null, null, AppOpsManager.OP_NONE, null, receivers,
                            null, 0, null, null, false, true, true, -1);
                    queue.enqueueParallelBroadcastLocked(r);
                    queue.scheduleBroadcastsLocked();
                }
            }

            return sticky;
        }
    }

    public void unregisterReceiver(IIntentReceiver receiver) {
        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Unregister receiver: " + receiver);

        final long origId = Binder.clearCallingIdentity();
        try {
            boolean doTrim = false;

            synchronized(this) {
                ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
                if (rl != null) {
                    final BroadcastRecord r = rl.curBroadcast;
                    if (r != null && r == r.queue.getMatchingOrderedReceiver(r)) {
                        final boolean doNext = r.queue.finishReceiverLocked(
                                r, r.resultCode, r.resultData, r.resultExtras,
                                r.resultAbort, false);
                        if (doNext) {
                            doTrim = true;
                            r.queue.processNextBroadcast(false);
                        }
                    }

                    if (rl.app != null) {
                        rl.app.receivers.remove(rl);
                    }
                    removeReceiverLocked(rl);
                    if (rl.linkedToDeath) {
                        rl.linkedToDeath = false;
                        rl.receiver.asBinder().unlinkToDeath(rl, 0);
                    }
                }
            }

            // If we actually concluded any broadcasts, we might now be able
            // to trim the recipients' apps from our working set
            if (doTrim) {
                trimApplications();
                return;
            }

        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
    ...
    
    public final int broadcastIntent(IApplicationThread caller,
            Intent intent, String resolvedType, IIntentReceiver resultTo,
            int resultCode, String resultData, Bundle resultExtras,
            String[] requiredPermissions, int appOp, Bundle bOptions,
            boolean serialized, boolean sticky, int userId) {
        enforceNotIsolatedCaller("broadcastIntent");
        synchronized(this) {
            intent = verifyBroadcastLocked(intent);

            final ProcessRecord callerApp = getRecordForAppLocked(caller);
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            int res = broadcastIntentLocked(callerApp,
                    callerApp != null ? callerApp.info.packageName : null,
                    intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
                    requiredPermissions, appOp, bOptions, serialized, sticky,
                    callingPid, callingUid, userId);
            Binder.restoreCallingIdentity(origId);
            return res;
        }
    }
    ...
    
```

**不管代码逻辑怎么变，他的传值都还是反射**

```
  ClassLoader cl =  mReceiver.getClass().getClassLoader();
                    intent.setExtrasClassLoader(cl);
                    intent.prepareToEnterProcess();
                    setExtrasClassLoader(cl);
                    receiver.setPendingResult(this);
                    receiver.onReceive(mContext, intent);
```


## 三、Binder

### 3.1 什么是IPC

IPC(Interprocess Communication)进程间通信

提供给程序员在不同进程之间的相互通信接口或技术

每个进程拥有0x00000000到0xFFFFFFFF的4G虚拟内存空间

由于进程的用户空间是相互独立的，不同进程不能相互访问，这是内核实现的保护机制

![MacDown logo](https://github.com/whsgzcy/AOSP7.1.1/blob/master/images/1.png)

UNIX中常见的IPC：
信号、信道、信号量、消息队列、共享存储、UNIX或套接字

信号：
信号是软件中断。他的本意是提供一种处理异步事件的方法。不同进程间通过信号来通知对方发生了某个事件，信号是IPC。

A进程直接向B进程发信号，B进程接收并处理信号

例：在终端键入ctrl+c发出sigint信号，进程接受处理这个信号，默认为终止。

管道：
无名管道，有名管道(FIFO)

管道的本质是读写“文件”，UNIX中万物都是文件。

无名管道，只能作用有亲缘的进程通信(父子进程，兄弟进程)，由于是无名管道，文件其实是在内存中的。

有名管道(FIFO)，也称为命名管道，可以在任意进程间通信，有实体的文件。

管道文件内的数据无论是内存中的文件还是实体文件，在读取后都会被清除，再次填入新数据。

常见的管道应用就是我们输入shell命令使用到的管道，例如：cat android.log|grep ViewRoot

信号量：本质只是一个锁，不具备数据交换的能力

消息队列：通信的双方通过再队列中存入消息和读取消息来进行通信

共享存储：在内存中开辟一块可被共享的内存，进程间就可以调用相应的函数访问共享内存，在共享内存的地方交换信息进行通信。


### 3.2 Android中的IPC Binder

**对象请求代理架构**

Android中展现给我们的是**Activity、Service、BroadCast、ContentProvider**四大组件，当不同进程中的组件进行数据交换时就需要进行间的通信，但从Android外的特性概念空间中，我们看不到进程的概念，而是**Activity、Service、AIDL、INTENT**。

我们想要请求音频焦点，首先需要**getSystemService(Activity.AUDIO_SERVICE)**就能够得到一个AudioManager对象，通过这个**AudioManager**对象就可以操作系统音频相关，完全看不到Binder，所以在Android中，要完成某个操作，所需要做的就是请求某个有能力的服务对象去完成，而无需知道这个通讯是怎样工作的，以及服务在哪里，Android的这种设计本质是一种对象请求代理架构，只需要请求得到一个远程对象的代理，通过这个代理就可以和远程对象通信。

![MacDown logo](https://github.com/whsgzcy/AOSP7.1.1/blob/master/images/2.png)


**Android的IPC原理**

每个Android的进程，只能运行在自己的进程所拥有的虚拟地址空间，对应4G的虚拟地址空间(32bit)，其中3G是用户空间，1G是内核空间，对于用户空间，不同进程之间彼此是不能共享的，而内核空间却是可以共享的，client进程向server进程通信，恰恰是利用进程间可共享的内核内存空间来完成底层通信工作的，client端与server端进程往往采用ioctl等方法跟内核空间驱动进行交互，而ioctl控制的就是Android中的Binder，Binder位于内核中，其本质只一个**驱动设备(/dev/binder)**

![MacDown logo](https://github.com/whsgzcy/AOSP7.1.1/blob/master/images/3.png)

**Binder原理**

Binder通信采用c/s架构，从组件视角来说，包含Client、Server、ServerManager以及binder驱动，架构图如下所示：

Client：我们的应用(内部拥有一个BpBinder)

Server：远程的服务(内部拥有一个BBinder)

ServiceManager：位于Native(c++)，用于管理系统中的各种服务内部维护一个服务列表service list

Binder驱动设备：位于内核中的Binder的本体，一个驱动设备

toctl：用来控制IO设备的系统函数

![MacDown logo](https://github.com/whsgzcy/AOSP7.1.1/blob/master/images/4.png)

图中client/server/serviceManager之间的相互通信都是基于Binder机制，有主要的三大步骤：

1、注册服务(addService)：server进程要先注册到ServiceManager，该过程：server是客户端，ServiceManager是服务端

2、获取服务(getService):client进程使用某个Service前，需向ServiceManager中获取对应的Service，该过程client是客户端，ServiceManager是服务端。

3、使用服务：Client根据得到的Service信息建立与Service所在Server进程通信的通路，然后可以直接与Service交互，该过程：client是客户端，server是服务端。

client、server、ServiceManager之间不是直接交互的，而是都通过与内核空间中的Binder进行交互的，从而实现IPC通信方式。

**数据通信流程**

从一般意义来讲，Android设计者在Linux内核中设计了一个叫做Binder的设备文件，专门用来进行Android的数据交换，所有从数据流来看Java的VM空间进入到c++空间进行了一次转换，并有c++空间的函数将转化过的对象通过driver\binder设备传递到服务进程，从而完成进程间的IPC，这个过程可以用下图来表示。

![MacDown logo](https://github.com/whsgzcy/AOSP7.1.1/blob/master/images/5.png)

这里的数据流有几层转换过程

(1)从JVM空间传到c++空间，这个是靠JNI使用ENV来完成对象的映射过程

(2)从c++空间传入内核Binder设备，使用processState类完成工作

(3)Service从内核中Binder设备读取数据

其中BpBinder(客户端)和BBinder(服务端)都是Android中Binder通信相关的代表，他们都从IBinder类中派生而来，关系如图

![MacDown logo](https://github.com/whsgzcy/AOSP7.1.1/blob/master/images/6.png)

### 3.3 获取服务流程

**获取代理对象**

IBinder b = ServiceManager.getService(Context.INPUT_METHOD_SERVICE);

IInputMethodManager.Stub.asInterface(b);

![MacDown logo](https://github.com/whsgzcy/AOSP7.1.1/blob/master/images/7.png)

[-> IServiceManager.cpp]

通过defaultServiceManager()得到BpServiceManager

interface_cast<IServiceManager>(ProcessState::self()->getContextObject(NULL));

1.ProcessState::self():

用于获取ProcessState对象(单例模式)

每个进程有且只有一个ProcessState对象，存在则直接返回，不存在则创建；

2.ProcessState::getContextObject():

用户获取BpBinder对象

对于handle=0的BpBinder对象(与ServiceManager建立通信)

存在则直接返回，不存在则创建

3.interface_cast<IServiceManager>(BpBinder):

用户获取BpServiceManagerd对象

通过BpServiceManager::getService()来获取服务

其中checkService()检索服务是否存在，当服务存在则直接返回相应服务，当服务不存在则休眠1s再继续检索服务，循环进行5次。checkService()中通过remote调用transact()方法来传递Service，remote()就是我们得到的BpBinder

其最终在IPCThreadState中进行真正的transact工作。

[->IPCThreadState.cpp]

每个线程都有一个IPCThreadState，其中有一个mOut，成员变量MProcess保存了ProcessState变量(每个进程只有一个)
mIn用来接收来自Binder设备的数据，默认大小256字节，mOut用来存储发往Binder设备的数据，默认大小为256字节。


问题2：

startService和bindService



参考：

https://blog.csdn.net/u013085697/article/details/53898879

https://blog.csdn.net/time_hunter/article/details/53107191

《深入理解Android卷III》张大伟著

https://v.qq.com/x/page/r0537eqpud6.html?spm=a2h7l.searchresults.soresults.dposter