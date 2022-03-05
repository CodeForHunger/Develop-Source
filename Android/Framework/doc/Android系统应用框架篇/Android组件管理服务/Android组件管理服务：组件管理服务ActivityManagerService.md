<h1 align="center">Android组件管理框架：组件管理服务ActivityManagerService</h1>

[toc]

ActivityManagerService是贯穿Android系统组件的核心服务，在ServiceServer执行run()方法的时候被创建，运行在独立的线程中，负责Activity、Service、BroadcastReceiver的启动、切换、调度以及应用进程的管理和调度工作。

Activity、Service、BroadcastReceiver的启动、切换、调度都有着相似的流程，我们来看一下。

Activity的启动流程图（放大可查看）如下所示：

<img src="https://github.com/BeesAndroid/BeesAndroid/raw/master/art/principle/server/activitymanagerservice/activity_start_flow.png" />

主要角色有：

- Instrumentation: 监控应用与系统相关的交互行为。
- AMS：组件管理调度中心，什么都不干，但是什么都管。
- ActivityStarter：处理Activity什么时候启动，怎么样启动相关问题，也就是处理Intent与Flag相关问题，平时提到的启动模式都可以在这里找到实现。
- ActivityStackSupervisior：这个类的作用你从它的名字就可以看出来，它用来管理Stack和Task。
- ActivityStack：用来管理栈里的Activity。
- ActivityThread：最终干活的人，是ActivityThread的内部类，Activity、Service、BroadcastReceiver的启动、切换、调度等各种操作都在这个类里完成。

Service的启动流程图（放大可查看）如下所示：

<img src="https://github.com/BeesAndroid/BeesAndroid/raw/master/art/principle/server/activitymanagerservice/service_start_flow.png" />

主要角色有：

- AMS：组件管理调度中心，什么都不干，但是什么都管。
- ApplicationThread：最终干活的人，是ActivityThread的内部类，Activity、Service、BroadcastReceiver的启动、切换、调度等各种操作都在这个类里完成。
- ActiveServices：主要用来管理Service，内部维护了三份列表：将启动Service列表、重启Service列表以及以销毁Service列表。

BroadcastReceiver的启动流程图（放大可查看）如下所示：

<img src="https://github.com/BeesAndroid/BeesAndroid/raw/master/art/principle/server/activitymanagerservice/broadcast_start_flow.png" />

主要角色有：

- AMS：组件管理调度中心，什么都不干，但是什么都管。
- BroadcastQueue：广播队列，根据广播的优先级来管理广播。
- ApplicationThread：最终干活的人，是ActivityThread的内部类，Activity、Service、BroadcastReceiver的启动、切换、调度等各种操作都在这个类里完成。
- ReceiverDispatcher：广播调度中心，采用反射的方式获取BroadcastReceiver的实例，然后调用它的onReceive()方法。

可以发现，除了一些辅助类外，最主要的组件管家AMS和应用主线程ActivityThread。本篇文章重点分析这两个类的实现，至于其他类会在
Activity、Service与BroadcastReceiver启动流程的文章中一一分析。

通过上面的分析，AMS的整个调度流程就非常明朗了。

>ActivityManager相当于前台接待，她将客户的各种需求传达给大总管ActivityMangerService，但是大总管自己不干活，他招来了很多小弟，他最信赖的小弟ActivityThread
替他完成真正的启动、切换、以及退出操作，至于其他的中间环节就交给ActivityStack、ActivityStarter等其他小弟来完成。

## 一 组件管家ActivityManagerService

### 1.1 ActivityManagerService启动流程

我们知道所有的系统服务都是在[SystemServer](https://android.googlesource.com/platform/frameworks/base/+/7d276c3/services/java/com/android/server/SystemServer.java)的run()方法里启动的，SystemServer
将系统服务分为了三类：

- 引导服务
- 核心服务
- 其他服务

ActivityManagerService属于引导服务，在startBootstrapServices()方法里被创建，如下所示：

```java
mActivityManagerService = mSystemServiceManager.startService(
        ActivityManagerService.Lifecycle.class).getService();
```
SystemServiceManager的startService()方法利用反射来创建对象，Lifecycle是ActivityManagerService里的静态内部类，它继承于SystemService，在它的构造方法里
它会调用ActivityManagerService的构造方法创建ActivityManagerService对象。

```java
public static final class Lifecycle extends SystemService {
    private final ActivityManagerService mService;

    public Lifecycle(Context context) {
        super(context);
        mService = new ActivityManagerService(context);
    }

    @Override
    public void onStart() {
        mService.start();
    }

    public ActivityManagerService getService() {
        return mService;
    }
}
```

ActivityManagerService的构造方法如下所示：

```java
public ActivityManagerService(Context systemContext) {
    mContext = systemContext;
    mFactoryTest = FactoryTest.getMode();
    mSystemThread = ActivityThread.currentActivityThread();

    Slog.i(TAG, "Memory class: " + ActivityManager.staticGetMemoryClass());

    //创建并启动系统线程以及相关Handler
    mHandlerThread = new ServiceThread(TAG,
            android.os.Process.THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
    mHandlerThread.start();
    mHandler = new MainHandler(mHandlerThread.getLooper());
    mUiHandler = new UiHandler();
    /* static; one-time init here */
    if (sKillHandler == null) {
        sKillThread = new ServiceThread(TAG + ":kill",
                android.os.Process.THREAD_PRIORITY_BACKGROUND, true /* allowIo */);
        sKillThread.start();
        sKillHandler = new KillHandler(sKillThread.getLooper());
    }

    //创建用来存储各种组件Activity、Broadcast的数据结构
    mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
            "foreground", BROADCAST_FG_TIMEOUT, false);
    mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
            "background", BROADCAST_BG_TIMEOUT, true);
    mBroadcastQueues[0] = mFgBroadcastQueue;
    mBroadcastQueues[1] = mBgBroadcastQueue;

    mServices = new ActiveServices(this);
    mProviderMap = new ProviderMap(this);
    mAppErrors = new AppErrors(mContext, this);

    //创建system等各种文件夹，用来记录系统的一些事件
    ...
    
    //初始化一些记录工具
    ...
}
```
可以发现，ActivityManagerService的构造方法主要做了两个事情：

- 创建并启动系统线程以及相关Handler。
- 创建用来存储各种组件Activity、Broadcast的数据结构。

这里有个问题，这里创建了两个Hanlder（sKillHandler暂时忽略，它是用来kill进程的）分别是MainHandler与UiHandler，它们有什么区别呢？🤔

我们知道Handler是用来向所在线程发送消息的，也就是说决定Handler定位的是它构造方法里的Looper，我们分别来看下。

MainHandler里的Looper来源于线程ServiceThread，它的线程名是"ActivityManagerService"，该Handler主要用来处理组件调度相关操作。

```java
mHandlerThread = new ServiceThread(TAG,
        android.os.Process.THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
mHandlerThread.start();
mHandler = new MainHandler(mHandlerThread.getLooper());
```

UiHandler里的Looper来源于线程UiThread（继承于ServiceThread），它的线程名"android.ui"，该Handler主要用来处理UI相关操作。

```java

private UiThread() {
    super("android.ui", Process.THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
    // Make sure UiThread is in the fg stune boost group
    Process.setThreadGroup(Process.myTid(), Process.THREAD_GROUP_TOP_APP);
}

public UiHandler() {
    super(com.android.server.UiThread.get().getLooper(), null, true);
}
```

以上便是整个ActivityManagerService的启动流程，还是比较简单的。

## 1.2 ActivityManagerService工作流程

[ActivityManagerService](https://android.googlesource.com/platform/frameworks/base/+/4f868ed/services/core/java/com/android/server/am/ActivityManagerService.java)就是ActivityManager家族
的核心类了，四大组件的启动、切换、调度都是在ActivityManagerService里完成的。

ActivityManagerService类图如下所示：

<img src="https://github.com/BeesAndroid/BeesAndroid/raw/master/art/principle/server/activitymanagerservice/activity_manager_service_class.png" />

- ActivityManager：AMS给客户端调用的接口。
- ActivityManagerNative：该类是ActivityManagerService的父类，继承与Binder，主要用来负责进程通信，接收ActivityManager传递过来的信息，这么写可以将通信部分分离在ActivityManagerNative，使得
ActivityManagerService可以专注组件的调度，减小了类的体积。
- ActivityManagerProxy：该类定义在ActivityManagerNative内部，正如它的名字那样，它是ActivityManagerService的代理类，

关于ActivityManager

>[ActivityManager](https://android.googlesource.com/platform/frameworks/base/+/742a67127366c376fdf188ff99ba30b27d3bf90c/core/java/android/app/ActivityManager.java)是提供给客户端调用的接口，日常开发中我们可以利用
ActivityManager来获取系统中正在运行的组件（Activity、Service）、进程（Process）、任务（Task）等信息，ActivityManager定义了相应的方法来获取和操作这些信息。

ActivityManager定义了很多静态内部类来描述这些信息，具体说来：

- ActivityManager.StackId： 描述组件栈ID信息
- ActivityManager.StackInfo： 描述组件栈信息，可以利用StackInfo去系统中检索某个栈。
- ActivityManager.MemoryInfo： 系统可用内存信息
- ActivityManager.RecentTaskInfo： 最近的任务信息
- ActivityManager.RunningAppProcessInfo： 正在运行的进程信息
- ActivityManager.RunningServiceInfo： 正在运行的服务信息
- ActivityManager.RunningTaskInfo： 正在运行的任务信息
- ActivityManager.AppTask： 描述应用任务信息

说道这里，我们有必要区分一些概念，以免以后混淆。

- 进程（Process）：Android系统进行资源调度和分配的基本单位，需要注意的是同一个栈的Activity可以运行在不同的进程里。
- 任务（Task）：Task是一组以栈的形式聚集在一起的Activity的集合，这个任务栈就是一个Task。
                      

在日常开发中，我们一般是不需要直接操作ActivityManager这个类，只有在一些特殊的开发场景才用的到。

- isLowRamDevice()：判断应用是否运行在一个低内存的Android设备上。
- clearApplicationUserData()：重置app里的用户数据。
- ActivityManager.AppTask/ActivityManager.RecentTaskInfo：我们如何需要操作Activity的栈信息也可以通过ActivityManager来做。

关于ActivityManagerNative与ActivityManagerProxy

>这两个类其实涉及的是Android的Binder通信原理，后面我们会有专门的文章来分析Binder相关实现。

### 1.3 ActivityManagerService组件信息管理

我们知道四大组件的启动依赖于进程，如果该进程没有启动，会先启动该进程，再进行attach，描述进程信息的是ProcessRecord，还有很多其他以Record结尾的类用来描述组件信息，如下所示：

<img src="https://github.com/BeesAndroid/BeesAndroid/raw/master/art/principle/server/activitymanagerservice/activity_,anager_service_reocrd_class.png" width="600"/>

- ProcessRecord：描述进程信息。
- ActivityRecord：描述Activity组件信息。
- ServiceRecord：描述Service组件信息。
- BroadcastRecord：描述Broadcast组件信息。
- ReceiverRecord：描述Broadcast Receiver信息。
- ContentProviderRecord：描述ContentProvider组件信息。
- ContentProviderConnection：描述ContentProviderConnection信息。

那么这些组件的信息都存储在哪里呢？🤔

- Activity的信息记录在ActivityStack、ActivityStackSupervisor和AM中。
- Service的信息记录在BroadcastQueue和AMS中。
- Broadcast的信息记录在ActiveServices和AMS中。
- Provider的信息记录在ProviderMap和AMS中。

