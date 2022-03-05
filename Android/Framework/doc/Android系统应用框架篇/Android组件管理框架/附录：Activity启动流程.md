<h1 align="center">附录：Activity启动流程</h1>

Activity启动流程序列图

<img src="../../../art/app/component/activity_start_sequence.png"/>

Activity启动流程结构图

<img src="../../../art/app/component/activity_start_structure.png"/>

### 1 Launcher.startActivitySafely(Intent intent, Object tag)

```java
/**
 * Default launcher application.
 */
public final class Launcher extends Activity
        implements View.OnClickListener, OnLongClickListener, LauncherModel.Callbacks, AllAppsView.Watcher {
        
    void startActivitySafely(Intent intent, Object tag) {
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        try {
            //调用Activity的startActivity()方法
            startActivity(intent);
        } catch (ActivityNotFoundException e) {
            Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
            Log.e(TAG, "Unable to launch. tag=" + tag + " intent=" + intent, e);
        } catch (SecurityException e) {
            Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
            Log.e(TAG, "Launcher does not have the permission to launch " + intent +
                    ". Make sure to create a MAIN intent-filter for the corresponding activity " +
                    "or use the exported attribute for this activity. "
                    + "tag="+ tag + " intent=" + intent, e);
        }
    }
    
}
```

我们来分析一下这段代码，当我们在手机上点击一个应用的图标时，上述代码就会运行。intent会先添加Intent.FLAG_ACTIVITY_NEW_TASK，保证新运行的应用
的Activity都建立在一个新的任务栈中。然后再去调用Activity的startActivity()方法。

那么应用的图标是怎么和这个应用的Lancher Activity联系起来的呢？

系统在启动的时候会启动PackageManagerService（包管理服务），所有的应用都是通过它安装的，PackageManagerService会对应用的AndroidManifest.xml
进行解析，从而得到应用里所有的组件信息。Lanucher组件在启动过程中就会去向PackageManagerService查询包含以下信息的Activity组件。

```xml
<intent-filter>
        <action android:text="android.intent.action.MAIN" />

        <category android:text="android.intent.category.LAUNCHER" />
    </intent-filter>
```

并为每一个包含该信息的Activity组件创建一个快捷图标，由此两者便建立了联系。关于Android应用的安装和启动流程，我们后续还有详细的文章做分析。


### 2 Activity.startActivity(Intent intent)

```java
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks {
        
        @Override
        public void startActivity(Intent intent) {
            startActivityForResult(intent, -1);
        }

}
```
startActivity()会去调用startActivityForResult(intent, -1)，-1表示不需要知道Activity的执行结果。

### 3 Activity.startActivityForResult(Intent intent, int requestCode)

```java
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks {
        
    public void startActivityForResult(Intent intent, int requestCode) {
        if (mParent == null) {
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }
        } else {
            mParent.startActivityFromChild(this, intent, requestCode);
        }
    }
}
```
该方法牵扯到了Activity内部的两个成员变量：

```
Intrumentation：用来监控应用与系统之间的交互操作。
```
Activity的启动也是一个应用与系统之间的交互操作，startActivityForResult()会去调用mInstrumentation.execStartActivity()来代为执行Activity的启动操
作，这样Intrumentation便可以去监控Activity的整个启动过程。

```
ActivityThread：用来描述一个应用进程。
```
每当系统启动一个应用进程时，都会启动一个ActivityThread实例，该实例保存在Activity的mThread变量中。


### 4 Instrumentation.execStartActivity(Context who, IBinder contextThread, IBinder token, Activity target, Intent intent, int requestCode)

```java
/**
 * Base class for implementing application instrumentation code.  When running
 * with instrumentation turned on, this class will be instantiated for you
 * before any of the application code, allowing you to monitor all of the
 * interaction the system has with the application.  An Instrumentation
 * implementation is described to the system through an AndroidManifest.xml's
 * &lt;instrumentation&gt; tag.
 */
public class Instrumentation {

    public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    if (am.match(who, null, intent)) {
                        am.mHits++;
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        null, 0, token, target != null ? target.mEmbeddedID : null,
                        requestCode, false, false);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
        }
        return null;
    }
    
}
```

我们再来看看传递给mInstrumentation.execStartActivity()方法的6个参数：

- Context who：被启动Activity的Context。
- IBinder contextThread：被启动Activity的主进程，由mMainThread.getApplicationThread()获取的一个类型为ApplicationThread
的Binder本地对象，该对象最终传递给了ActivityManagerService。
- IBinder token：启动该Activity的内部token，供系统识别。它其实是一个代理对象，指向了ActivityManagerService中一个类型为ActivityRecord的
Binder本地对象。
- Activity target：调用startActivity的tartget Activity。
- Intent intent：intent
- int requestCode：requestCode

Instrumentation.execStartActivity()先调用ActivityManagerNative.getDefault()获取ActivityManagerService的代理对象，再调用ActivityManagerService
.startActivity()启动Activity。

我们先来看一下getDefault()方法的实现。

```java
public abstract class ActivityManagerNative extends Binder implements IActivityManager{

      private static IActivityManager gDefault;

      /**
       * Cast a Binder object into an application thread interface, generating
       * a proxy if needed.
       */
      static public IApplicationThread asInterface(IBinder obj) {
          if (obj == null) {
              return null;
          }
          IApplicationThread in =
              (IApplicationThread)obj.queryLocalInterface(descriptor);
          if (in != null) {
              return in;
          }
          
          return new ApplicationThreadProxy(obj);
      }
    
      /**
       * Retrieve the system's default/global activity manager.
       */
      static public IActivityManager getDefault()
      {
          if (gDefault != null) {
              //if (Config.LOGV) Log.v(
              //    "ActivityManager", "returning cur default = " + gDefault);
              return gDefault;
          }
          IBinder b = ServiceManager.getService("activity");
          if (Config.LOGV) Log.v(
              "ActivityManager", "default service binder = " + b);
          gDefault = asInterface(b);
          if (Config.LOGV) Log.v(
              "ActivityManager", "default service = " + gDefault);
          return gDefault;
      }  
}
  
```
ActivityManagerNative.getDefault()通过ServiceManager.getService("activity")获取一个引用ActivityManagerService的服务代理对象，然后
调用asInterface(b)函数将其封装成一个类型为ApplicationThreadProxy的代理对象，并保存在gDefault静态变量中。ApplicationThreadProxy实现了
IActivityManager里的相关方法。

### 5 ApplicationThreadProxy.startActivity(IApplicationThread caller, Intent intent, String resolvedType, Uri[] grantedUriPermissions, int grantedMode, IBinder resultTo, String resultWho, int requestCode, boolean onlyIfNeeded, boolean debug)

```java
class ActivityManagerProxy implements IActivityManager{

    private IBinder mRemote;

    public int startActivity(IApplicationThread caller, Intent intent,
            String resolvedType, Uri[] grantedUriPermissions, int grantedMode,
            IBinder resultTo, String resultWho,
            int requestCode, boolean onlyIfNeeded,
            boolean debug) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeTypedArray(grantedUriPermissions, 0);
        data.writeInt(grantedMode);
        data.writeStrongBinder(resultTo);
        data.writeString(resultWho);
        data.writeInt(requestCode);
        data.writeInt(onlyIfNeeded ? 1 : 0);
        data.writeInt(debug ? 1 : 0);
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
    
}
```
我们先来看看主要的参数

- IApplicationThread caller：指向Launcher组件所运行在的应用进程的ApplicationThread对象。
- Intent intent：将要启动的Activity组件的信息。
- IBinder resultTo：指向ActivityMangerService内部的一个ActivityRecord对象，它保存了Launcher组件的详细信息。

ActivityManagerProxy.startActivity()将传递过来的参数写入Parcel对象中，并通过ActivityManagerProxy内部的Binder对象mRemote发起一个
类型为START_ACTIVITY_TRANSACTION的进程间通信请求。


### 6 ActivityManagerService.startActivity(IApplicationThread caller, Intent intent, String resolvedType, Uri[] grantedUriPermissions, int grantedMode, IBinder resultTo, String resultWho, int requestCode, boolean onlyIfNeeded, boolean debug)

```java

public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

    public final int startActivity(IApplicationThread caller,
            Intent intent, String resolvedType, Uri[] grantedUriPermissions,
            int grantedMode, IBinder resultTo,
            String resultWho, int requestCode, boolean onlyIfNeeded,
            boolean debug) {
        return mMainStack.startActivityMayWait(caller, intent, resolvedType,
                grantedUriPermissions, grantedMode, resultTo, resultWho,
                requestCode, onlyIfNeeded, debug, null, null);
    }
        
}
```

ActivityManagerService.startActivity()接收ActivityManagerProxy传递过来的START_ACTIVITY_TRANSACTION的进程间通信请求，进一步调用ActivityStack.startActivityMayWait()
去处理。

>ActivityStack：用来描述Activity组件的堆栈，

### 7 ActivityStack.startActivityMayWait(IApplicationThread caller, Intent intent, String resolvedType, Uri[] grantedUriPermissions, int grantedMode, IBinder resultTo, String resultWho, int requestCode, boolean onlyIfNeeded,  boolean debug, WaitResult outResult, Configuration config)
```java
public class ActivityStack {

     final int startActivityMayWait(IApplicationThread caller,
                Intent intent, String resolvedType, Uri[] grantedUriPermissions,
                int grantedMode, IBinder resultTo,
                String resultWho, int requestCode, boolean onlyIfNeeded,
                boolean debug, WaitResult outResult, Configuration config) {
            // Refuse possible leaked file descriptors
            if (intent != null && intent.hasFileDescriptors()) {
                throw new IllegalArgumentException("File descriptors passed in Intent");
            }
    
            boolean componentSpecified = intent.getComponent() != null;
            
            // Don't modify the client's object!
            intent = new Intent(intent);
    
            //到PackageManagerService里去解析intent里的信息，并保存到ActivityInfo中。
            // Collect information about the target of the Intent.
            ActivityInfo aInfo;
            try {
                ResolveInfo rInfo =
                    AppGlobals.getPackageManager().resolveIntent(
                            intent, resolvedType,
                            PackageManager.MATCH_DEFAULT_ONLY
                            | ActivityManagerService.STOCK_PM_FLAGS);
                aInfo = rInfo != null ? rInfo.activityInfo : null;
            } catch (RemoteException e) {
                aInfo = null;
            }
    
            if (aInfo != null) {
                // Store the found target back into the intent, because now that
                // we have it we never want to do this again.  For example, if the
                // user navigates back to this point in the history, we should
                // always restart the exact same activity.
                intent.setComponent(new ComponentName(
                        aInfo.applicationInfo.packageName, aInfo.text));
    
                // Don't debug things in the system process
                if (debug) {
                    if (!aInfo.processName.equals("system")) {
                        mService.setDebugApp(aInfo.processName, true, false);
                    }
                }
            }
    
            synchronized (mService) {
                int callingPid;
                int callingUid;
                if (caller == null) {
                    callingPid = Binder.getCallingPid();
                    callingUid = Binder.getCallingUid();
                } else {
                    callingPid = callingUid = -1;
                }
                
                mConfigWillChange = config != null
                        && mService.mConfiguration.diff(config) != 0;
                if (DEBUG_CONFIGURATION) Slog.v(TAG,
                        "Starting activity when config will change = " + mConfigWillChange);
                
                final long origId = Binder.clearCallingIdentity();
                
                if (mMainStack && aInfo != null &&
                        (aInfo.applicationInfo.flags&ApplicationInfo.FLAG_CANT_SAVE_STATE) != 0) {
                    // This may be a heavy-weight process!  Check to see if we already
                    // have another, different heavy-weight process running.
                    if (aInfo.processName.equals(aInfo.applicationInfo.packageName)) {
                        if (mService.mHeavyWeightProcess != null &&
                                (mService.mHeavyWeightProcess.info.uid != aInfo.applicationInfo.uid ||
                                !mService.mHeavyWeightProcess.processName.equals(aInfo.processName))) {
                            int realCallingPid = callingPid;
                            int realCallingUid = callingUid;
                            if (caller != null) {
                                ProcessRecord callerApp = mService.getRecordForAppLocked(caller);
                                if (callerApp != null) {
                                    realCallingPid = callerApp.pid;
                                    realCallingUid = callerApp.info.uid;
                                } else {
                                    Slog.w(TAG, "Unable to find app for caller " + caller
                                          + " (pid=" + realCallingPid + ") when starting: "
                                          + intent.toString());
                                    return START_PERMISSION_DENIED;
                                }
                            }
                            
                            IIntentSender target = mService.getIntentSenderLocked(
                                    IActivityManager.INTENT_SENDER_ACTIVITY, "android",
                                    realCallingUid, null, null, 0, intent,
                                    resolvedType, PendingIntent.FLAG_CANCEL_CURRENT
                                    | PendingIntent.FLAG_ONE_SHOT);
                            
                            Intent newIntent = new Intent();
                            if (requestCode >= 0) {
                                // Caller is requesting a result.
                                newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_HAS_RESULT, true);
                            }
                            newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_INTENT,
                                    new IntentSender(target));
                            if (mService.mHeavyWeightProcess.activities.size() > 0) {
                                ActivityRecord hist = mService.mHeavyWeightProcess.activities.get(0);
                                newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_CUR_APP,
                                        hist.packageName);
                                newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_CUR_TASK,
                                        hist.task.taskId);
                            }
                            newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_NEW_APP,
                                    aInfo.packageName);
                            newIntent.setFlags(intent.getFlags());
                            newIntent.setClassName("android",
                                    HeavyWeightSwitcherActivity.class.getName());
                            intent = newIntent;
                            resolvedType = null;
                            caller = null;
                            callingUid = Binder.getCallingUid();
                            callingPid = Binder.getCallingPid();
                            componentSpecified = true;
                            try {
                                ResolveInfo rInfo =
                                    AppGlobals.getPackageManager().resolveIntent(
                                            intent, null,
                                            PackageManager.MATCH_DEFAULT_ONLY
                                            | ActivityManagerService.STOCK_PM_FLAGS);
                                aInfo = rInfo != null ? rInfo.activityInfo : null;
                            } catch (RemoteException e) {
                                aInfo = null;
                            }
                        }
                    }
                }
                
                //调用startActivityLocked()启动Activity组件
                int res = startActivityLocked(caller, intent, resolvedType,
                        grantedUriPermissions, grantedMode, aInfo,
                        resultTo, resultWho, requestCode, callingPid, callingUid,
                        onlyIfNeeded, componentSpecified);
                
                if (mConfigWillChange && mMainStack) {
                    // If the caller also wants to switch to a new configuration,
                    // do so now.  This allows a clean switch, as we are waiting
                    // for the current activity to pause (so we will not destroy
                    // it), and have not yet started the next activity.
                    mService.enforceCallingPermission(android.Manifest.permission.CHANGE_CONFIGURATION,
                            "updateConfiguration()");
                    mConfigWillChange = false;
                    if (DEBUG_CONFIGURATION) Slog.v(TAG,
                            "Updating to new configuration after starting activity.");
                    mService.updateConfigurationLocked(config, null);
                }
                
                Binder.restoreCallingIdentity(origId);
                
                if (outResult != null) {
                    outResult.result = res;
                    if (res == IActivityManager.START_SUCCESS) {
                        mWaitingActivityLaunched.add(outResult);
                        do {
                            try {
                                mService.wait();
                            } catch (InterruptedException e) {
                            }
                        } while (!outResult.timeout && outResult.who == null);
                    } else if (res == IActivityManager.START_TASK_TO_FRONT) {
                        ActivityRecord r = this.topRunningActivityLocked(null);
                        if (r.nowVisible) {
                            outResult.timeout = false;
                            outResult.who = new ComponentName(r.info.packageName, r.info.text);
                            outResult.totalTime = 0;
                            outResult.thisTime = 0;
                        } else {
                            outResult.thisTime = SystemClock.uptimeMillis();
                            mWaitingActivityVisible.add(outResult);
                            do {
                                try {
                                    mService.wait();
                                } catch (InterruptedException e) {
                                }
                            } while (!outResult.timeout && outResult.who == null);
                        }
                    }
                }
                
                return res;
            }
        }

}
```

这个函数代码比较多，但就做了两件事情：

```
1 到PackageManagerService里去解析intent里的信息，并保存到ActivityInfo中。
2 调用startActivityLocked()启动Activity组件
```

### 8 ActivityStack.startActivityLocked(IApplicationThread caller, Intent intent, String resolvedType, Uri[] grantedUriPermissions, int grantedMode, ActivityInfo aInfo, IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid, boolean onlyIfNeeded, boolean componentSpecified)

```java
public class ActivityStack {

     final int startActivityLocked(IApplicationThread caller,
                Intent intent, String resolvedType,
                Uri[] grantedUriPermissions,
                int grantedMode, ActivityInfo aInfo, IBinder resultTo,
                String resultWho, int requestCode,
                int callingPid, int callingUid, boolean onlyIfNeeded,
                boolean componentSpecified) {
    
            int err = START_SUCCESS;
    
            ProcessRecord callerApp = null;
            if (caller != null) {
                //获取ProcessRecord对象，它实际指向了Launcher组件所运行的应用进程
                callerApp = mService.getRecordForAppLocked(caller);
                if (callerApp != null) {
                    //进程的pid与uid
                    callingPid = callerApp.pid;
                    callingUid = callerApp.info.uid;
                } else {
                    Slog.w(TAG, "Unable to find app for caller " + caller
                          + " (pid=" + callingPid + ") when starting: "
                          + intent.toString());
                    err = START_PERMISSION_DENIED;
                }
            }
    
            if (err == START_SUCCESS) {
                Slog.i(TAG, "Starting: " + intent + " from pid "
                        + (callerApp != null ? callerApp.pid : callingPid));
            }
    
            ActivityRecord sourceRecord = null;
            ActivityRecord resultRecord = null;
            if (resultTo != null) {
                //指向ActivityMangerService内部的一个ActivityRecord对象，它保存了Launcher组件的详细信息。
                int index = indexOfTokenLocked(resultTo);
                if (DEBUG_RESULTS) Slog.v(
                    TAG, "Sending result to " + resultTo + " (index " + index + ")");
                if (index >= 0) {
                    //在Activity组件堆栈中找到用来描述Launcher组件的一个ActivityRecord对象，并保存到sourceRecord变量中。
                    sourceRecord = (ActivityRecord)mHistory.get(index);
                    if (requestCode >= 0 && !sourceRecord.finishing) {
                        resultRecord = sourceRecord;
                    }
                }
            }
    
            int launchFlags = intent.getFlags();
    
            if ((launchFlags&Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0
                    && sourceRecord != null) {
                // Transfer the result target from the source activity to the new
                // one being started, including any failures.
                if (requestCode >= 0) {
                    return START_FORWARD_AND_REQUEST_CONFLICT;
                }
                resultRecord = sourceRecord.resultTo;
                resultWho = sourceRecord.resultWho;
                requestCode = sourceRecord.requestCode;
                sourceRecord.resultTo = null;
                if (resultRecord != null) {
                    resultRecord.removeResultsLocked(
                        sourceRecord, resultWho, requestCode);
                }
            }
    
            if (err == START_SUCCESS && intent.getComponent() == null) {
                // We couldn't find a class that can handle the given Intent.
                // That's the end of that!
                err = START_INTENT_NOT_RESOLVED;
            }
    
            if (err == START_SUCCESS && aInfo == null) {
                // We couldn't find the specific class specified in the Intent.
                // Also the end of the line.
                err = START_CLASS_NOT_FOUND;
            }
    
            if (err != START_SUCCESS) {
                if (resultRecord != null) {
                    sendActivityResultLocked(-1,
                        resultRecord, resultWho, requestCode,
                        Activity.RESULT_CANCELED, null);
                }
                return err;
            }
    
            final int perm = mService.checkComponentPermission(aInfo.permission, callingPid,
                    callingUid, aInfo.exported ? -1 : aInfo.applicationInfo.uid);
            if (perm != PackageManager.PERMISSION_GRANTED) {
                if (resultRecord != null) {
                    sendActivityResultLocked(-1,
                        resultRecord, resultWho, requestCode,
                        Activity.RESULT_CANCELED, null);
                }
                String msg = "Permission Denial: starting " + intent.toString()
                        + " from " + callerApp + " (pid=" + callingPid
                        + ", uid=" + callingUid + ")"
                        + " requires " + aInfo.permission;
                Slog.w(TAG, msg);
                throw new SecurityException(msg);
            }
    
            if (mMainStack) {
                if (mService.mController != null) {
                    boolean abort = false;
                    try {
                        // The Intent we give to the watcher has the extra data
                        // stripped off, since it can contain private information.
                        Intent watchIntent = intent.cloneFilter();
                        abort = !mService.mController.activityStarting(watchIntent,
                                aInfo.applicationInfo.packageName);
                    } catch (RemoteException e) {
                        mService.mController = null;
                    }
        
                    if (abort) {
                        if (resultRecord != null) {
                            sendActivityResultLocked(-1,
                                resultRecord, resultWho, requestCode,
                                Activity.RESULT_CANCELED, null);
                        }
                        // We pretend to the caller that it was really started, but
                        // they will just get a cancel result.
                        return START_SUCCESS;
                    }
                }
            }
            
            //创建一个ActivityRecord对象，用来描述即将启动的Activity组件。
            ActivityRecord r = new ActivityRecord(mService, this, callerApp, callingUid,
                    intent, resolvedType, aInfo, mService.mConfiguration,
                    resultRecord, resultWho, requestCode, componentSpecified);
    
            if (mMainStack) {
                if (mResumedActivity == null
                        || mResumedActivity.info.applicationInfo.uid != callingUid) {
                    if (!mService.checkAppSwitchAllowedLocked(callingPid, callingUid, "Activity start")) {
                        PendingActivityLaunch pal = new PendingActivityLaunch();
                        pal.r = r;
                        pal.sourceRecord = sourceRecord;
                        pal.grantedUriPermissions = grantedUriPermissions;
                        pal.grantedMode = grantedMode;
                        pal.onlyIfNeeded = onlyIfNeeded;
                        mService.mPendingActivityLaunches.add(pal);
                        return START_SWITCHES_CANCELED;
                    }
                }
            
                if (mService.mDidAppSwitch) {
                    // This is the second allowed switch since we stopped switches,
                    // so now just generally allow switches.  Use case: user presses
                    // home (switches disabled, switch to home, mDidAppSwitch now true);
                    // user taps a home icon (coming from home so allowed, we hit here
                    // and now allow anyone to switch again).
                    mService.mAppSwitchesAllowedTime = 0;
                } else {
                    mService.mDidAppSwitch = true;
                }
             
                mService.doPendingActivityLaunchesLocked(false);
            }
            
            return startActivityUncheckedLocked(r, sourceRecord,
                    grantedUriPermissions, grantedMode, onlyIfNeeded, true);
        }
}
```

又是一个长方法，莫慌O(∩_∩)O，方法虽长，理清逻辑就能很清晰的读懂这些方法了。写来看看上述方法里出现的几个成员变量。


```
ProcessRecord callerApp：每个应用进程都用一个ProcessRecord来描述，该对象保存在ActivityManagerService，它实际指向了Launcher组件所运行的应用进程。
ActivityManagerService mService：ActivityManagerService
IApplicationThread caller：指向的是Launcher组件所运行在的应用进程中的一个ApplicationThread对象。

IBinder resultTo：指向ActivityMangerService内部的一个ActivityRecord对象，它保存了Launcher组件的详细信息。
ActivityStack mHistory：用来描述系统的Activity堆栈。堆栈中的Activity对象都由ActivityRecord对象来描述。
ActivityRecord sourceRecord：执行启动Activity组件操作的Launcher信息。
ActivityRecord r：执行启动Activity组件操作的目标Activity信息。
```

了解了参数的意义，我们来总结一下该函数做了哪些事情。

```
1 获取Launcher组件所运行的应用进程信息，它保存在了ProcessRecord callerApp中。
2 在Activity组件堆栈中找到用来描述Launcher组件的一个ActivityRecord对象，并保存到sourceRecord变量中。sourceRecord描述了执行Activity启动操作的Launcher信息。
3 创建ActivityRecord r，它描述了执行启动Activity组件操作的目标Activity信息。
4 以上述创建的变量为参数，调用startActivityUncheckedLocked()，进一步执行启动操作。
```

### 9 ActivityStack.startActivityUncheckedLocked(ActivityRecord r, ActivityRecord sourceRecord, Uri[] grantedUriPermissions, int grantedMode, boolean onlyIfNeeded, boolean doResume) 

```java
public class ActivityStack {

     final int startActivityUncheckedLocked(ActivityRecord r,
                ActivityRecord sourceRecord, Uri[] grantedUriPermissions,
                int grantedMode, boolean onlyIfNeeded, boolean doResume) {
            final Intent intent = r.intent;
            final int callingUid = r.launchedFromUid;
            
            //获取Activity启动标志位，保存在launchFlags中
            int launchFlags = intent.getFlags();
            
            //一个按位与操作，检测launchFlags的Intent.FLAG_ACTIVITY_NO_USER_ACTION位是否等于1，
            // We'll invoke onUserLeaving before onPause only if the launching
            // activity did not explicitly state that this is an automated launch.
            mUserLeaving = (launchFlags&Intent.FLAG_ACTIVITY_NO_USER_ACTION) == 0;
            if (DEBUG_USER_LEAVING) Slog.v(TAG,
                    "startActivity() => mUserLeaving=" + mUserLeaving);
            
            // If the caller has asked not to resume at this point, we make note
            // of this in the record so that we can skip it when trying to find
            // the top running activity.
            if (!doResume) {
                r.delayedResume = true;
            }
            
            ActivityRecord notTop = (launchFlags&Intent.FLAG_ACTIVITY_PREVIOUS_IS_TOP)
                    != 0 ? r : null;
    
            // If the onlyIfNeeded flag is set, then we can do this if the activity
            // being launched is the same as the one making the call...  or, as
            // a special case, if we do not know the caller then we count the
            // current top activity as the caller.
            if (onlyIfNeeded) {
                ActivityRecord checkedCaller = sourceRecord;
                if (checkedCaller == null) {
                    checkedCaller = topRunningNonDelayedActivityLocked(notTop);
                }
                if (!checkedCaller.realActivity.equals(r.realActivity)) {
                    // Caller is not the same as launcher, so always needed.
                    onlyIfNeeded = false;
                }
            }
    
            if (sourceRecord == null) {
                // This activity is not being started from another...  in this
                // case we -always- start a new task.
                if ((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
                    Slog.w(TAG, "startActivity called from non-Activity context; forcing Intent.FLAG_ACTIVITY_NEW_TASK for: "
                          + intent);
                    launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
                }
            } else if (sourceRecord.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
                // The original activity who is starting us is running as a single
                // instance...  this new activity it is starting must go on its
                // own task.
                launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
            } else if (r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE
                    || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK) {
                // The activity being started is a single instance...  it always
                // gets launched into its own task.
                launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
            }
    
            if (r.resultTo != null && (launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
                // For whatever reason this activity is being launched into a new
                // task...  yet the caller has requested a result back.  Well, that
                // is pretty messed up, so instead immediately send back a cancel
                // and let the new task continue launched as normal without a
                // dependency on its originator.
                Slog.w(TAG, "Activity is launching as a new task, so cancelling activity result.");
                sendActivityResultLocked(-1,
                        r.resultTo, r.resultWho, r.requestCode,
                    Activity.RESULT_CANCELED, null);
                r.resultTo = null;
            }
    
            //addingToTask初始化为false，表示要为目标Activity创建一个专属任务。
            boolean addingToTask = false;
            
            //检查这个专属任务是否已经存在，如果存在则addingToTask置为true
            if (((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0 &&
                    (launchFlags&Intent.FLAG_ACTIVITY_MULTIPLE_TASK) == 0)
                    || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK
                    || r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
                // If bring to front is requested, and no result is requested, and
                // we can find a task that was started with this same
                // component, then instead of launching bring that one to the front.
                if (r.resultTo == null) {
                    // See if there is a task to bring to the front.  If this is
                    // a SINGLE_INSTANCE activity, there can be one and only one
                    // instance of it in the history, and it is always in its own
                    // unique task, so we do a special search.
                    ActivityRecord taskTop = r.launchMode != ActivityInfo.LAUNCH_SINGLE_INSTANCE
                            ? findTaskLocked(intent, r.info)
                            : findActivityLocked(intent, r.info);
                    if (taskTop != null) {
                        if (taskTop.task.intent == null) {
                            // This task was started because of movement of
                            // the activity based on affinity...  now that we
                            // are actually launching it, we can assign the
                            // base intent.
                            taskTop.task.setIntent(intent, r.info);
                        }
                        // If the target task is not in the front, then we need
                        // to bring it to the front...  except...  well, with
                        // SINGLE_TASK_LAUNCH it's not entirely clear.  We'd like
                        // to have the same behavior as if a new instance was
                        // being started, which means not bringing it to the front
                        // if the caller is not itself in the front.
                        ActivityRecord curTop = topRunningNonDelayedActivityLocked(notTop);
                        if (curTop.task != taskTop.task) {
                            r.intent.addFlags(Intent.FLAG_ACTIVITY_BROUGHT_TO_FRONT);
                            boolean callerAtFront = sourceRecord == null
                                    || curTop.task == sourceRecord.task;
                            if (callerAtFront) {
                                // We really do want to push this one into the
                                // user's face, right now.
                                moveTaskToFrontLocked(taskTop.task, r);
                            }
                        }
                        // If the caller has requested that the target task be
                        // reset, then do so.
                        if ((launchFlags&Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                            taskTop = resetTaskIfNeededLocked(taskTop, r);
                        }
                        if (onlyIfNeeded) {
                            // We don't need to start a new activity, and
                            // the client said not to do anything if that
                            // is the case, so this is it!  And for paranoia, make
                            // sure we have correctly resumed the top activity.
                            if (doResume) {
                                resumeTopActivityLocked(null);
                            }
                            return START_RETURN_INTENT_TO_CALLER;
                        }
                        if ((launchFlags&Intent.FLAG_ACTIVITY_CLEAR_TOP) != 0
                                || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK
                                || r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
                            // In this situation we want to remove all activities
                            // from the task up to the one being started.  In most
                            // cases this means we are resetting the task to its
                            // initial state.
                            ActivityRecord top = performClearTaskLocked(
                                    taskTop.task.taskId, r, launchFlags, true);
                            if (top != null) {
                                if (top.frontOfTask) {
                                    // Activity aliases may mean we use different
                                    // intents for the top activity, so make sure
                                    // the task now has the identity of the new
                                    // intent.
                                    top.task.setIntent(r.intent, r.info);
                                }
                                logStartActivity(EventLogTags.AM_NEW_INTENT, r, top.task);
                                top.deliverNewIntentLocked(callingUid, r.intent);
                            } else {
                                // A special case: we need to
                                // start the activity because it is not currently
                                // running, and the caller has asked to clear the
                                // current task to have this activity at the top.
                                addingToTask = true;
                                // Now pretend like this activity is being started
                                // by the top of its task, so it is put in the
                                // right place.
                                sourceRecord = taskTop;
                            }
                        } else if (r.realActivity.equals(taskTop.task.realActivity)) {
                            // In this case the top activity on the task is the
                            // same as the one being launched, so we take that
                            // as a request to bring the task to the foreground.
                            // If the top activity in the task is the root
                            // activity, deliver this new intent to it if it
                            // desires.
                            if ((launchFlags&Intent.FLAG_ACTIVITY_SINGLE_TOP) != 0
                                    && taskTop.realActivity.equals(r.realActivity)) {
                                logStartActivity(EventLogTags.AM_NEW_INTENT, r, taskTop.task);
                                if (taskTop.frontOfTask) {
                                    taskTop.task.setIntent(r.intent, r.info);
                                }
                                taskTop.deliverNewIntentLocked(callingUid, r.intent);
                            } else if (!r.intent.filterEquals(taskTop.task.intent)) {
                                // In this case we are launching the root activity
                                // of the task, but with a different intent.  We
                                // should start a new instance on top.
                                addingToTask = true;
                                sourceRecord = taskTop;
                            }
                        } else if ((launchFlags&Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) == 0) {
                            // In this case an activity is being launched in to an
                            // existing task, without resetting that task.  This
                            // is typically the situation of launching an activity
                            // from a notification or shortcut.  We want to place
                            // the new activity on top of the current task.
                            addingToTask = true;
                            sourceRecord = taskTop;
                        } else if (!taskTop.task.rootWasReset) {
                            // In this case we are launching in to an existing task
                            // that has not yet been started from its front door.
                            // The current task has been brought to the front.
                            // Ideally, we'd probably like to place this new task
                            // at the bottom of its stack, but that's a little hard
                            // to do with the current organization of the code so
                            // for now we'll just drop it.
                            taskTop.task.setIntent(r.intent, r.info);
                        }
                        if (!addingToTask) {
                            // We didn't do anything...  but it was needed (a.k.a., client
                            // don't use that intent!)  And for paranoia, make
                            // sure we have correctly resumed the top activity.
                            if (doResume) {
                                resumeTopActivityLocked(null);
                            }
                            return START_TASK_TO_FRONT;
                        }
                    }
                }
            }
    
            //String uri = r.intent.toURI();
            //Intent intent2 = new Intent(uri);
            //Slog.i(TAG, "Given intent: " + r.intent);
            //Slog.i(TAG, "URI is: " + uri);
            //Slog.i(TAG, "To intent: " + intent2);
    
            if (r.packageName != null) {
                // If the activity being launched is the same as the one currently
                // at the top, then we need to check if it should only be launched
                // once.
                ActivityRecord top = topRunningNonDelayedActivityLocked(notTop);
                if (top != null && r.resultTo == null) {
                    if (top.realActivity.equals(r.realActivity)) {
                        if (top.app != null && top.app.thread != null) {
                            if ((launchFlags&Intent.FLAG_ACTIVITY_SINGLE_TOP) != 0
                                || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TOP
                                || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK) {
                                logStartActivity(EventLogTags.AM_NEW_INTENT, top, top.task);
                                // For paranoia, make sure we have correctly
                                // resumed the top activity.
                                if (doResume) {
                                    resumeTopActivityLocked(null);
                                }
                                if (onlyIfNeeded) {
                                    // We don't need to start a new activity, and
                                    // the client said not to do anything if that
                                    // is the case, so this is it!
                                    return START_RETURN_INTENT_TO_CALLER;
                                }
                                top.deliverNewIntentLocked(callingUid, r.intent);
                                return START_DELIVERED_TO_TOP;
                            }
                        }
                    }
                }
    
            } else {
                if (r.resultTo != null) {
                    sendActivityResultLocked(-1,
                            r.resultTo, r.resultWho, r.requestCode,
                        Activity.RESULT_CANCELED, null);
                }
                return START_CLASS_NOT_FOUND;
            }
    
            boolean newTask = false;
    
            // Should this be considered a new task?
            //目标Activity是否要在新进程中启动
            if (r.resultTo == null && !addingToTask
                    && (launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
                // todo: should do better management of integers.
                mService.mCurTask++;
                if (mService.mCurTask <= 0) {
                    mService.mCurTask = 1;
                }
                //创建专属任务，并保存在r.task中。newTask置为true
                r.task = new TaskRecord(mService.mCurTask, r.info, intent,
                        (r.info.flags&ActivityInfo.FLAG_CLEAR_TASK_ON_LAUNCH) != 0);
                if (DEBUG_TASKS) Slog.v(TAG, "Starting new activity " + r
                        + " in new task " + r.task);
                newTask = true;
                if (mMainStack) {
                    //新建的专属任务交由ActivityManagerService处理
                    mService.addRecentTaskLocked(r.task);
                }
                
            } else if (sourceRecord != null) {
                if (!addingToTask &&
                        (launchFlags&Intent.FLAG_ACTIVITY_CLEAR_TOP) != 0) {
                    // In this case, we are adding the activity to an existing
                    // task, but the caller has asked to clear that task if the
                    // activity is already running.
                    ActivityRecord top = performClearTaskLocked(
                            sourceRecord.task.taskId, r, launchFlags, true);
                    if (top != null) {
                        logStartActivity(EventLogTags.AM_NEW_INTENT, r, top.task);
                        top.deliverNewIntentLocked(callingUid, r.intent);
                        // For paranoia, make sure we have correctly
                        // resumed the top activity.
                        if (doResume) {
                            resumeTopActivityLocked(null);
                        }
                        return START_DELIVERED_TO_TOP;
                    }
                } else if (!addingToTask &&
                        (launchFlags&Intent.FLAG_ACTIVITY_REORDER_TO_FRONT) != 0) {
                    // In this case, we are launching an activity in our own task
                    // that may already be running somewhere in the history, and
                    // we want to shuffle it to the front of the stack if so.
                    int where = findActivityInHistoryLocked(r, sourceRecord.task.taskId);
                    if (where >= 0) {
                        ActivityRecord top = moveActivityToFrontLocked(where);
                        logStartActivity(EventLogTags.AM_NEW_INTENT, r, top.task);
                        top.deliverNewIntentLocked(callingUid, r.intent);
                        if (doResume) {
                            resumeTopActivityLocked(null);
                        }
                        return START_DELIVERED_TO_TOP;
                    }
                }
                // An existing activity is starting this new activity, so we want
                // to keep the new one in the same task as the one that is starting
                // it.
                r.task = sourceRecord.task;
                if (DEBUG_TASKS) Slog.v(TAG, "Starting new activity " + r
                        + " in existing task " + r.task);
    
            } else {
                //已经存在的Activity，直接将其移至栈顶即可
                // This not being started from an existing activity, and not part
                // of a new task...  just put it in the top task, though these days
                // this case should never happen.
                final int N = mHistory.size();
                ActivityRecord prev =
                    N > 0 ? (ActivityRecord)mHistory.get(N-1) : null;
                r.task = prev != null
                    ? prev.task
                    : new TaskRecord(mService.mCurTask, r.info, intent,
                            (r.info.flags&ActivityInfo.FLAG_CLEAR_TASK_ON_LAUNCH) != 0);
                if (DEBUG_TASKS) Slog.v(TAG, "Starting new activity " + r
                        + " in new guessed " + r.task);
            }
    
            if (grantedUriPermissions != null && callingUid > 0) {
                for (int i=0; i<grantedUriPermissions.length; i++) {
                    mService.grantUriPermissionLocked(callingUid, r.packageName,
                            grantedUriPermissions[i], grantedMode, r.getUriPermissionsLocked());
                }
            }
    
            mService.grantUriPermissionFromIntentLocked(callingUid, r.packageName,
                    intent, r.getUriPermissionsLocked());
    
            if (newTask) {
                EventLog.writeEvent(EventLogTags.AM_CREATE_TASK, r.task.taskId);
            }
            logStartActivity(EventLogTags.AM_CREATE_ACTIVITY, r, r.task);
            //调用重载函数startActivityLocked(r, newTask, doResume)进一步执行Activity启动操作
            startActivityLocked(r, newTask, doResume);
            return START_SUCCESS;
        }
 
}
```

我们先来看一下这个函数的参数

ActivityRecord r：用来描述将要启动的目标Activity。
ctivityRecord sourceRecord：用来描述发起启动操作的源Activity。

我们来分析一下这个函数做了那个事情。

1 获取intent里的flag，并保存在launchFlags中，从最开始的调用我们知道，此时launchFlags里只有Intent.FLAG_ACTIVITY_NEW_TASK被设置为1，其他位均为0.
然后再去检查Intent.FLAG_ACTIVITY_NO_USER_ACTION是否为1：

如果为1：则说明目标Activity不是用户手动启动的。
如果为0：则说明目标Activity是用户手动启动的，mUserLeaving置为true，表示后面要向Launcher发送一个用户离开的通知。

2 检查Activity的启动模式与启动标志位，最终决定Activity会被分配到那个任务栈中，下面分析一下具体的判断流程。

如果Intent.FLAG_ACTIVITY_NEW_TASK被设置为1，且Launcher不需要知道目标Activity的运行结果，那么ActivityManagerService就会将目标Activity运行在
另一个任务栈里。

三个问题：

如果判定将要启动的目标Activity是否要运行在一个新创建的进程里？

```
是否需要创建新进程，取决于两个因素：
1 PackageManagerService为应用分配的用户ID。
2 Activity组件的android;processs属性。
如果这个属性同在原有的进程里没有匹配的就会创建新进程。
```

除了手动添加Intent.FLAG_ACTIVITY_NEW_TASK，哪些情况下还会自动添加Intent.FLAG_ACTIVITY_NEW_TASK？

```
以下情况下，会自动向launchFlags中通添加Intent.FLAG_ACTIVITY_NEW_TASK。
启动Activity的Context是一个non-activity的Context
launchMode为ActivityInfo.LAUNCH_SINGLE_INSTANCE、ActivityInfo.LAUNCH_SINGLE_TASK、ActivityInfo.LAUNCH_SINGLE_INSTANCE
```

>注：Intent的flag采用非常巧妙的十六进制表示法，每个位置上包含一个flag，关于flag的详细信息可以参见[Android系统应用框架篇：Activity启动模式与标识位]()。

任务栈是如果被创建的？

```
ActivityManagerService在为目标Activity创建的任务栈有可能是一个新的任务栈也有可能是已经存在的任务在，这取决于Activity的android:taskAffinity属性。
该属性描述了Activity的专属任务。如果该专属任务已经存在，则将目标Activity添加到该任务中，如果不存在，则先去创建该专属任务，再将目标Activity添加到该
任务中。
```

我们再来来看看代码里具体是怎么实现。

```
1 addingToTask置为false，addingToTask初始化为false，检查这个专属任务是否已经存在，如果存在则addingToTask置为true
2 创建专属任务，并保存在r.task中。newTask置为true，新建的专属任务交由ActivityManagerService处理。
```
3 调用重载函数startActivityLocked(r, newTask, doResume)进一步执行Activity启动操作。 

```java
public class ActivityStack {

      private final void startActivityLocked(ActivityRecord r, boolean newTask,
                boolean doResume) {
                
            //开始对目标Activity进行入栈操作
            final int NH = mHistory.size();
    
            int addPos = -1;
            
            if (!newTask) {
                //如果目标Activity是一个已经存在的任务，则在mHistory栈里进行查找
                // If starting in an existing task, find where that is...
                boolean startIt = true;
                for (int i = NH-1; i >= 0; i--) {
                
                    //自上而下搜索Activity任务栈，找到可以允许目标Activity的任务
                    //接着再将这个任务栈最上面的Activity的位置i加1，作为目标Activity
                    //在mmHistory栈中的位置。
                    ActivityRecord p = (ActivityRecord)mHistory.get(i);
                    if (p.finishing) {
                        continue;
                    }
                    if (p.task == r.task) {
                        //查找到了目标Activity，如果它当前不可见，就先添加到mHistory栈里
                        //不做启动，当用户点击返回，目标Activity重新可见时，再启动目标Activity
                        // Here it is!  Now, if this is not yet visible to the
                        // user, then just add it without starting; it will
                        // get started when the user navigates back to it.
                        addPos = i+1;
                        if (!startIt) {
                            mHistory.add(addPos, r);
                            r.inHistory = true;
                            r.task.numActivities++;
                            mService.mWindowManager.addAppToken(addPos, r, r.task.taskId,
                                    r.info.screenOrientation, r.fullscreen);
                            if (VALIDATE_TOKENS) {
                                mService.mWindowManager.validateAppTokens(mHistory);
                            }
                            return;
                        }
                        break;
                    }
                    if (p.fullscreen) {
                        startIt = false;
                    }
                }
            }
    
            //如果目标Activity在一个新的任务中启动时，即newTask=true，这时ActivityStack就会将该目标Activity放在栈顶
            //addPos=NH=mHistory.size()
            // Place a new activity at top of stack, so it is next to interact
            // with the user.
            if (addPos < 0) {
                addPos = NH;
            }
            
            // If we are not placing the new activity frontmost, we do not want
            // to deliver the onUserLeaving callback to the actual frontmost
            // activity
            if (addPos < NH) {
                mUserLeaving = false;
                if (DEBUG_USER_LEAVING) Slog.v(TAG, "startActivity() behind front, mUserLeaving=false");
            }
            
            // Slot the activity into the history stack and proceed
            mHistory.add(addPos, r);
            r.inHistory = true;
            r.frontOfTask = newTask;
            r.task.numActivities++;
            if (NH > 0) {
                // We want to show the starting preview window if we are
                // switching to a new task, or the next activity's process is
                // not currently running.
                boolean showStartingIcon = newTask;
                ProcessRecord proc = r.app;
                if (proc == null) {
                    proc = mService.mProcessNames.get(r.processName, r.info.applicationInfo.uid);
                }
                if (proc == null || proc.thread == null) {
                    showStartingIcon = true;
                }
                if (DEBUG_TRANSITION) Slog.v(TAG,
                        "Prepare open transition: starting " + r);
                if ((r.intent.getFlags()&Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {
                    mService.mWindowManager.prepareAppTransition(WindowManagerPolicy.TRANSIT_NONE);
                    mNoAnimActivities.add(r);
                } else if ((r.intent.getFlags()&Intent.FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET) != 0) {
                    mService.mWindowManager.prepareAppTransition(
                            WindowManagerPolicy.TRANSIT_TASK_OPEN);
                    mNoAnimActivities.remove(r);
                } else {
                    mService.mWindowManager.prepareAppTransition(newTask
                            ? WindowManagerPolicy.TRANSIT_TASK_OPEN
                            : WindowManagerPolicy.TRANSIT_ACTIVITY_OPEN);
                    mNoAnimActivities.remove(r);
                }
                mService.mWindowManager.addAppToken(
                        addPos, r, r.task.taskId, r.info.screenOrientation, r.fullscreen);
                boolean doShow = true;
                if (newTask) {
                    // Even though this activity is starting fresh, we still need
                    // to reset it to make sure we apply affinities to move any
                    // existing activities from other tasks in to it.
                    // If the caller has requested that the target task be
                    // reset, then do so.
                    if ((r.intent.getFlags()
                            &Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                        resetTaskIfNeededLocked(r, r);
                        doShow = topRunningNonDelayedActivityLocked(null) == r;
                    }
                }
                if (SHOW_APP_STARTING_PREVIEW && doShow) {
                    // Figure out if we are transitioning from another activity that is
                    // "has the same starting icon" as the next one.  This allows the
                    // window manager to keep the previous window it had previously
                    // created, if it still had one.
                    ActivityRecord prev = mResumedActivity;
                    if (prev != null) {
                        // We don't want to reuse the previous starting preview if:
                        // (1) The current activity is in a different task.
                        if (prev.task != r.task) prev = null;
                        // (2) The current activity is already displayed.
                        else if (prev.nowVisible) prev = null;
                    }
                    mService.mWindowManager.setAppStartingWindow(
                            r, r.packageName, r.theme, r.nonLocalizedLabel,
                            r.labelRes, r.icon, prev, showStartingIcon);
                }
            } else {
                // If this is the first activity, don't do any fancy animations,
                // because there is nothing for it to animate on top of.
                mService.mWindowManager.addAppToken(addPos, r, r.task.taskId,
                        r.info.screenOrientation, r.fullscreen);
            }
            if (VALIDATE_TOKENS) {
                mService.mWindowManager.validateAppTokens(mHistory);
            }
    
            if (doResume) {
                //激活mHistory栈顶的Activity，也就是我们的目标Activity。
                resumeTopActivityLocked(null);
            }
        }
    
}
```

第8步也有个startActivityLocked函数：

```
ActivityStack.startActivityLocked(IApplicationThread caller, Intent intent, String resolvedType, Uri[] grantedUriPermissions, int grantedMode, ActivityInfo aInfo, IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid, boolean onlyIfNeeded, boolean componentSpecified)

```

这个是它的重载版本，它主要做了以下事情：

```
1 开始对目标Activity进行入栈操作，如果目标Activity是一个已经存在的任务，则在mHistory栈里进行查找，查找到了目标Activity，如果它当前不可
见，就先添加到mHistory栈里不做启动，当用户点击返回，目标Activity重新可见时，再启动目标Activity。
2 如果目标Activity在一个新的任务中启动时，即newTask=true，这时ActivityStack就会将该目标Activity放在栈顶addPos=NH=mHistory.size()
3 调用resumeTopActivityLocked(null)，激活mHistory栈顶的Activity，也就是我们的目标Activity。
```

### 10 ActivityStack.resumeTopActivityLocked(ActivityRecord prev) 

```java
public class ActivityStack {

     /**
         * Ensure that the top activity in the stack is resumed.
         *
         * @param prev The previously resumed activity, for when in the process
         * of pausing; can be null to call from elsewhere.
         *
         * @return Returns true if something is being resumed, or false if
         * nothing happened.
         */
        final boolean resumeTopActivityLocked(ActivityRecord prev) {
            //查找当前Activity栈最上面一个不是处于结束状态的Activity组件。在上一步我们
            //将目标Activity添加到栈顶，所以这个next指向的也就是将要启动的Activity组件，
            // Find the first activity that is not finishing.
            ActivityRecord next = topRunningActivityLocked(null);
    
            //userLeaving用来表示是否需要向Activity组件发送一个用户离开事件。
            // Remember how we'll process this pause/resume situation, and ensure
            // that the state is reset however we wind up proceeding.
            final boolean userLeaving = mUserLeaving;
            mUserLeaving = false;
    
            if (next == null) {
                //没有更多Activity，返回桌面。
                // There are no more activities!  Let's just start up the
                // Launcher...
                if (mMainStack) {
                    return mService.startHomeActivityLocked();
                }
            }
    
            next.delayedResume = false;
            
            //如果要启动的Activity组件next等于当前被激活的Activity，且处于ActivityState.RESUMED状态，则直接返回，
            // If the top activity is the resumed one, nothing to do.
            if (mResumedActivity == next && next.state == ActivityState.RESUMED) {
                // Make sure we have executed any pending transitions, since there
                // should be nothing left to do at this point.
                mService.mWindowManager.executeAppTransition();
                mNoAnimActivities.clear();
                return false;
            }
    
            //如果要启动的Activity组件next等于等于上次被终止的Activity组件或者系统正在休眠或关机，则直接返回。
            // If we are sleeping, and there is no resumed activity, and the top
            // activity is paused, well that is the state we want.
            if ((mService.mSleeping || mService.mShuttingDown)
                    && mLastPausedActivity == next && next.state == ActivityState.PAUSED) {
                // Make sure we have executed any pending transitions, since there
                // should be nothing left to do at this point.
                mService.mWindowManager.executeAppTransition();
                mNoAnimActivities.clear();
                return false;
            }
            
            // The activity may be waiting for stop, but that is no longer
            // appropriate for it.
            mStoppingActivities.remove(next);
            mWaitingVisibleActivities.remove(next);
    
            if (DEBUG_SWITCH) Slog.v(TAG, "Resuming " + next);
    
            //如果当前我们正在终止一个Activity组件，则直接返回，等待它终止完成，这也说明了Activity生命周期方法里是
            //旧Activity先onPause(),新Activity再onResume()
            // If we are currently pausing an activity, then don't do anything
            // until that is done.
            if (mPausingActivity != null) {
                if (DEBUG_SWITCH) Slog.v(TAG, "Skip resume: pausing=" + mPausingActivity);
                return false;
            }
    
            // Okay we are now going to start a switch, to 'next'.  We may first
            // have to pause the current activity, but this is an important point
            // where we have decided to go to 'next' so keep track of that.
            // XXX "App Redirected" dialog is getting too many false positives
            // at this point, so turn off for now.
            if (false) {
                if (mLastStartedActivity != null && !mLastStartedActivity.finishing) {
                    long now = SystemClock.uptimeMillis();
                    final boolean inTime = mLastStartedActivity.startTime != 0
                            && (mLastStartedActivity.startTime + START_WARN_TIME) >= now;
                    final int lastUid = mLastStartedActivity.info.applicationInfo.uid;
                    final int nextUid = next.info.applicationInfo.uid;
                    if (inTime && lastUid != nextUid
                            && lastUid != next.launchedFromUid
                            && mService.checkPermission(
                                    android.Manifest.permission.STOP_APP_SWITCHES,
                                    -1, next.launchedFromUid)
                            != PackageManager.PERMISSION_GRANTED) {
                        mService.showLaunchWarningLocked(mLastStartedActivity, next);
                    } else {
                        next.startTime = now;
                        mLastStartedActivity = next;
                    }
                } else {
                    next.startTime = SystemClock.uptimeMillis();
                    mLastStartedActivity = next;
                }
            }
            
            //暂停当前Activity，是栈顶Activity进入resume状态
            // We need to start pausing the current activity so the top one
            // can be resumed...
            if (mResumedActivity != null) {
                if (DEBUG_SWITCH) Slog.v(TAG, "Skip resume: need to start pausing");
                startPausingLocked(userLeaving, false);
                return true;
            }
    
            if (prev != null && prev != next) {
                if (!prev.waitingVisible && next != null && !next.nowVisible) {
                    prev.waitingVisible = true;
                    mWaitingVisibleActivities.add(prev);
                    if (DEBUG_SWITCH) Slog.v(
                            TAG, "Resuming top, waiting visible to hide: " + prev);
                } else {
                    // The next activity is already visible, so hide the previous
                    // activity's windows right now so we can show the new one ASAP.
                    // We only do this if the previous is finishing, which should mean
                    // it is on top of the one being resumed so hiding it quickly
                    // is good.  Otherwise, we want to do the normal route of allowing
                    // the resumed activity to be shown so we can decide if the
                    // previous should actually be hidden depending on whether the
                    // new one is found to be full-screen or not.
                    if (prev.finishing) {
                        mService.mWindowManager.setAppVisibility(prev, false);
                        if (DEBUG_SWITCH) Slog.v(TAG, "Not waiting for visible to hide: "
                                + prev + ", waitingVisible="
                                + (prev != null ? prev.waitingVisible : null)
                                + ", nowVisible=" + next.nowVisible);
                    } else {
                        if (DEBUG_SWITCH) Slog.v(TAG, "Previous already visible but still waiting to hide: "
                            + prev + ", waitingVisible="
                            + (prev != null ? prev.waitingVisible : null)
                            + ", nowVisible=" + next.nowVisible);
                    }
                }
            }
    
            // We are starting up the next activity, so tell the window manager
            // that the previous one will be hidden soon.  This way it can know
            // to ignore it when computing the desired screen orientation.
            if (prev != null) {
                if (prev.finishing) {
                    if (DEBUG_TRANSITION) Slog.v(TAG,
                            "Prepare close transition: prev=" + prev);
                    if (mNoAnimActivities.contains(prev)) {
                        mService.mWindowManager.prepareAppTransition(WindowManagerPolicy.TRANSIT_NONE);
                    } else {
                        mService.mWindowManager.prepareAppTransition(prev.task == next.task
                                ? WindowManagerPolicy.TRANSIT_ACTIVITY_CLOSE
                                : WindowManagerPolicy.TRANSIT_TASK_CLOSE);
                    }
                    mService.mWindowManager.setAppWillBeHidden(prev);
                    mService.mWindowManager.setAppVisibility(prev, false);
                } else {
                    if (DEBUG_TRANSITION) Slog.v(TAG,
                            "Prepare open transition: prev=" + prev);
                    if (mNoAnimActivities.contains(next)) {
                        mService.mWindowManager.prepareAppTransition(WindowManagerPolicy.TRANSIT_NONE);
                    } else {
                        mService.mWindowManager.prepareAppTransition(prev.task == next.task
                                ? WindowManagerPolicy.TRANSIT_ACTIVITY_OPEN
                                : WindowManagerPolicy.TRANSIT_TASK_OPEN);
                    }
                }
                if (false) {
                    mService.mWindowManager.setAppWillBeHidden(prev);
                    mService.mWindowManager.setAppVisibility(prev, false);
                }
            } else if (mHistory.size() > 1) {
                if (DEBUG_TRANSITION) Slog.v(TAG,
                        "Prepare open transition: no previous");
                if (mNoAnimActivities.contains(next)) {
                    mService.mWindowManager.prepareAppTransition(WindowManagerPolicy.TRANSIT_NONE);
                } else {
                    mService.mWindowManager.prepareAppTransition(WindowManagerPolicy.TRANSIT_ACTIVITY_OPEN);
                }
            }
    
            if (next.app != null && next.app.thread != null) {
                if (DEBUG_SWITCH) Slog.v(TAG, "Resume running: " + next);
    
                // This activity is now becoming visible.
                mService.mWindowManager.setAppVisibility(next, true);
    
                ActivityRecord lastResumedActivity = mResumedActivity;
                ActivityState lastState = next.state;
    
                mService.updateCpuStats();
                
                next.state = ActivityState.RESUMED;
                mResumedActivity = next;
                next.task.touchActiveTime();
                mService.updateLruProcessLocked(next.app, true, true);
                updateLRUListLocked(next);
    
                // Have the window manager re-evaluate the orientation of
                // the screen based on the new activity order.
                boolean updated = false;
                if (mMainStack) {
                    synchronized (mService) {
                        Configuration config = mService.mWindowManager.updateOrientationFromAppTokens(
                                mService.mConfiguration,
                                next.mayFreezeScreenLocked(next.app) ? next : null);
                        if (config != null) {
                            next.frozenBeforeDestroy = true;
                        }
                        updated = mService.updateConfigurationLocked(config, next);
                    }
                }
                if (!updated) {
                    // The configuration update wasn't able to keep the existing
                    // instance of the activity, and instead started a new one.
                    // We should be all done, but let's just make sure our activity
                    // is still at the top and schedule another run if something
                    // weird happened.
                    ActivityRecord nextNext = topRunningActivityLocked(null);
                    if (DEBUG_SWITCH) Slog.i(TAG,
                            "Activity config changed during resume: " + next
                            + ", new next: " + nextNext);
                    if (nextNext != next) {
                        // Do over!
                        mHandler.sendEmptyMessage(RESUME_TOP_ACTIVITY_MSG);
                    }
                    if (mMainStack) {
                        mService.setFocusedActivityLocked(next);
                    }
                    ensureActivitiesVisibleLocked(null, 0);
                    mService.mWindowManager.executeAppTransition();
                    mNoAnimActivities.clear();
                    return true;
                }
                
                try {
                    // Deliver all pending results.
                    ArrayList a = next.results;
                    if (a != null) {
                        final int N = a.size();
                        if (!next.finishing && N > 0) {
                            if (DEBUG_RESULTS) Slog.v(
                                    TAG, "Delivering results to " + next
                                    + ": " + a);
                            next.app.thread.scheduleSendResult(next, a);
                        }
                    }
    
                    if (next.newIntents != null) {
                        next.app.thread.scheduleNewIntent(next.newIntents, next);
                    }
    
                    EventLog.writeEvent(EventLogTags.AM_RESUME_ACTIVITY,
                            System.identityHashCode(next),
                            next.task.taskId, next.shortComponentName);
                    
                    next.app.thread.scheduleResumeActivity(next,
                            mService.isNextTransitionForward());
                    
                    pauseIfSleepingLocked();
    
                } catch (Exception e) {
                    // Whoops, need to restart this activity!
                    next.state = lastState;
                    mResumedActivity = lastResumedActivity;
                    Slog.i(TAG, "Restarting because process died: " + next);
                    if (!next.hasBeenLaunched) {
                        next.hasBeenLaunched = true;
                    } else {
                        if (SHOW_APP_STARTING_PREVIEW && mMainStack) {
                            mService.mWindowManager.setAppStartingWindow(
                                    next, next.packageName, next.theme,
                                    next.nonLocalizedLabel,
                                    next.labelRes, next.icon, null, true);
                        }
                    }
                    startSpecificActivityLocked(next, true, false);
                    return true;
                }
    
                // From this point on, if something goes wrong there is no way
                // to recover the activity.
                try {
                    next.visible = true;
                    completeResumeLocked(next);
                } catch (Exception e) {
                    // If any exception gets thrown, toss away this
                    // activity and try the next one.
                    Slog.w(TAG, "Exception thrown during resume of " + next, e);
                    requestFinishActivityLocked(next, Activity.RESULT_CANCELED, null,
                            "resume-exception");
                    return true;
                }
    
                // Didn't need to use the icicle, and it is now out of date.
                next.icicle = null;
                next.haveState = false;
                next.stopped = false;
    
            } else {
                // Whoops, need to restart this activity!
                if (!next.hasBeenLaunched) {
                    next.hasBeenLaunched = true;
                } else {
                    if (SHOW_APP_STARTING_PREVIEW) {
                        mService.mWindowManager.setAppStartingWindow(
                                next, next.packageName, next.theme,
                                next.nonLocalizedLabel,
                                next.labelRes, next.icon, null, true);
                    }
                    if (DEBUG_SWITCH) Slog.v(TAG, "Restarting: " + next);
                }
                startSpecificActivityLocked(next, true, true);
            }
    
            return true;
        }
        
}
```
ActivityStack里有3个成员变量：

```
mResumedActivity：系统当前激活的Activity组件。
mLastPausedActivity：上一次被终止的Activity组件。
mLastPausingActivity：正在被终止的Activity组件。
```

好，我们来看看这个函数做了哪些事情：

1 查找当前Activity栈最上面一个不是处于结束状态的Activity组件。在上一步我们将目标Activity添加到栈顶，所以这个next指向的也就是将要启动的Activity组件，

2 检查其他Activity状态以及系统状态，遇到以下情况，函数会直接返回：

```
1 如果要启动的Activity组件next等于当前被激活的Activity，且处于ActivityState.RESUMED状态，则直接返回，
2 如果要启动的Activity组件next等于等于上次被终止的Activity组件或者系统正在休眠或关机，则直接返回。
3 如果当前我们正在终止一个Activity组件，则直接返回，等待它终止完成，这也说明了Activity生命周期方法里是旧Activity先onPause(),新Activity再onResume()
```

3 调用startPausingLocked(userLeaving, false)，暂停当前Activity，是栈顶Activity进入resume状态。

### 11 ActivityStack.startPausingLocked(boolean userLeaving, boolean uiSleeping)

```java
public class ActivityStack {
    
    private final void startPausingLocked(boolean userLeaving, boolean uiSleeping) {
            if (mPausingActivity != null) {
                RuntimeException e = new RuntimeException();
                Slog.e(TAG, "Trying to pause when pause is already pending for "
                      + mPausingActivity, e);
            }
            //prev指向即将进入Paused状态的Launcher组件
            ActivityRecord prev = mResumedActivity;
            if (prev == null) {
                RuntimeException e = new RuntimeException();
                Slog.e(TAG, "Trying to pause when nothing is resumed", e);
                resumeTopActivityLocked(null);
                return;
            }
            if (DEBUG_PAUSE) Slog.v(TAG, "Start pausing: " + prev);
            mResumedActivity = null;
            mPausingActivity = prev;
            mLastPausedActivity = prev;
            prev.state = ActivityState.PAUSING;
            prev.task.touchActiveTime();
    
            mService.updateCpuStats();
            
            if (prev.app != null && prev.app.thread != null) {
                if (DEBUG_PAUSE) Slog.v(TAG, "Enqueueing pending pause: " + prev);
                try {
                    EventLog.writeEvent(EventLogTags.AM_PAUSE_ACTIVITY,
                            System.identityHashCode(prev),
                            prev.shortComponentName);
                    //ActivityManagerService给Launcher所在的应用进程发送一个终止Launcher组件的通知
                    //以便Launcher可以执行一些数据保存操作。
                    prev.app.thread.schedulePauseActivity(prev, prev.finishing, userLeaving,
                            prev.configChangeFlags);
                    if (mMainStack) {
                        mService.updateUsageStats(prev, false);
                    }
                } catch (Exception e) {
                    // Ignore exception, if process died other code will cleanup.
                    Slog.w(TAG, "Exception thrown during pause", e);
                    mPausingActivity = null;
                    mLastPausedActivity = null;
                }
            } else {
                mPausingActivity = null;
                mLastPausedActivity = null;
            }
    
            // If we are not going to sleep, we want to ensure the device is
            // awake until the next activity is started.
            if (!mService.mSleeping && !mService.mShuttingDown) {
                mLaunchingActivity.acquire();
                if (!mHandler.hasMessages(LAUNCH_TIMEOUT_MSG)) {
          
                    // To be safe, don't allow the wake lock to be held for too long.
                    Message msg = mHandler.obtainMessage(LAUNCH_TIMEOUT_MSG);
                    mHandler.sendMessageDelayed(msg, LAUNCH_TIMEOUT);
                }
            }
    
    
            if (mPausingActivity != null) {
                // Have the window manager pause its key dispatching until the new
                // activity has started.  If we're pausing the activity just because
                // the screen is being turned off and the UI is sleeping, don't interrupt
                // key dispatch; the same activity will pick it up again on wakeup.
                if (!uiSleeping) {
                    prev.pauseKeyDispatchingLocked();
                } else {
                    if (DEBUG_PAUSE) Slog.v(TAG, "Key dispatch not paused for screen off");
                }
    
                //Launcher处理完ActivityManagerService给它发送的通知后，会再次向目标ActivityManagerService
                //所运行的线程的消息队列中发送一个PAUSE_TIMEOUT_MSG，该消息会在PAUSE_TIMEOUT毫秒后被处理。
                // Schedule a pause timeout in case the app doesn't respond.
                // We don't give it much time because this directly impacts the
                // responsiveness seen by the user.
                Message msg = mHandler.obtainMessage(PAUSE_TIMEOUT_MSG);
                msg.obj = prev;
                mHandler.sendMessageDelayed(msg, PAUSE_TIMEOUT);
                if (DEBUG_PAUSE) Slog.v(TAG, "Waiting for pause to complete...");
            } else {
                // This activity failed to schedule the
                // pause, so just treat it as being paused now.
                if (DEBUG_PAUSE) Slog.v(TAG, "Activity not running, resuming next.");
                resumeTopActivityLocked(null);
            }
        }
    
}
```

先来说说上述函数新出现的几个成员变量的含义：

```
ActivityRecord.app：类型为ProcessRecord，用来描述一个Activity所在应用进程的信息。
ProcessRecord.thread：类型为ApplicationThreadProxy，用来描述一个Binder的代理对象，引用的是一个类型为ApplicationThread的Binder本地对象。
```

再来说说这个函数做了哪些事情：

```
1 处理变量，将ActivityRecord prev、mPausingActivity与mLastPausedActivity指向即将进入Paused状态的Launcher组件，并将mResumedActivity置为null。
2 调用ApplicationThreadProxy。schedulePauseActivity(prev, prev.finishing, userLeaving, prev.configChangeFlags)给Launcher所在的应用进程发送一个终止Launcher组件的通知以便Launcher可以执行一些数据保存操作。
3 Launcher处理完ActivityManagerService给它发送的通知后，会再次向目标ActivityManagerService所运行的线程的消息队列中发送一个PAUSE_TIMEOUT_MSG，该消息会在PAUSE_TIMEOUT毫秒后被处理。
```

我们再来看看ApplicationThreadProxy。schedulePauseActivity(prev, prev.finishing, userLeaving, prev.configChangeFlags)的实现。

### 12 ApplicationThreadProxy。schedulePauseActivity(prev, prev.finishing, userLeaving, prev.configChangeFlags)

```java
class ApplicationThreadProxy implements IApplicationThread {

        public final void schedulePauseActivity(IBinder token, boolean finished,
                boolean userLeaving, int configChanges) throws RemoteException {
            Parcel data = Parcel.obtain();
            data.writeInterfaceToken(IApplicationThread.descriptor);
            data.writeStrongBinder(token);
            data.writeInt(finished ? 1 : 0);
            data.writeInt(userLeaving ? 1 :0);
            data.writeInt(configChanges);
            mRemote.transact(SCHEDULE_PAUSE_ACTIVITY_TRANSACTION, data, null,
                    IBinder.FLAG_ONEWAY);
            data.recycle();
        }

}
```

```
mRemote：ApplicationThreadProxy内部的一个Binder代理对象，用来做跨进程通信。
```

将传递过来的数据写入到Parcel对象中，并用mRemote向Launcher所在进程发送一个类型为SCHEDULE_PAUSE_ACTIVITY_TRANSACTION的进程间通信请求
，该请求是异步的，ActivityManagerService发送了该请求后立即返回。

### 13 ActivityThread.schedulePauseActivity(IBinder token, boolean finished, boolean userLeaving, int configChanges)

```java

public final class ActivityThread {

    public final void schedulePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges) {
        queueOrSendMessage(
                finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
                token,
                (userLeaving ? 1 : 0),
                configChanges);
    }

}
```
我们先来看看该函数的参数含义：

```
boolean finished:传递过来的值为false。
boolean userLeaving：传递过来的值为true。
IBinder token：Binder代理对象，指向了ActivityManagerService中与Launcher对应的一个ActivityRecordd对象。
```

schedulePauseActivity()会调用queueOrSendMessage()准备向Launcher所在的主线程消息队列发送一个类型为PAUSE_ACTIVITY的消息。

### 14 ActivityThread.queueOrSendMessage(int what, Object obj, int arg1, int arg2)

```java

public final class ActivityThread {

    final H mH = new H();
    
    private final void queueOrSendMessage(int what, Object obj, int arg1, int arg2) {
        synchronized (this) {
            if (DEBUG_MESSAGES) Slog.v(
                TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
                + ": " + arg1 + " / " + obj);
            Message msg = Message.obtain();
            msg.what = what;
            msg.obj = obj;
            msg.arg1 = arg1;
            msg.arg2 = arg2;
            mH.sendMessage(msg);
        }
    }

}
```
```
H：继承于Handler，用来发送和处理消息。
```

将schedulePauseActivity()传递过来的参数包装成一个Message对象，将消息PAUSE_ACTIVITY发送给Launcher的主线程的消息队列。

### 15 H.handleMessage(Message msg)

```java
private final class H extends Handler{

     public void handleMessage(Message msg) {
                if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + msg.what);
                switch (msg.what) {
                    case LAUNCH_ACTIVITY: {
                        ActivityClientRecord r = (ActivityClientRecord)msg.obj;
    
                        r.packageInfo = getPackageInfoNoCheck(
                                r.activityInfo.applicationInfo);
                        handleLaunchActivity(r, null);
                    } break;
                    case RELAUNCH_ACTIVITY: {
                        ActivityClientRecord r = (ActivityClientRecord)msg.obj;
                        handleRelaunchActivity(r, msg.arg1);
                    } break;
                    //处理PAUSE_ACTIVITY消息
                    case PAUSE_ACTIVITY:
                        handlePauseActivity((IBinder)msg.obj, false, msg.arg1 != 0, msg.arg2);
                        maybeSnapshot();
                        break;
                    case PAUSE_ACTIVITY_FINISHING:
                        handlePauseActivity((IBinder)msg.obj, true, msg.arg1 != 0, msg.arg2);
                        break;
                    case STOP_ACTIVITY_SHOW:
                        handleStopActivity((IBinder)msg.obj, true, msg.arg2);
                        break;
                    case STOP_ACTIVITY_HIDE:
                        handleStopActivity((IBinder)msg.obj, false, msg.arg2);
                        break;
                    case SHOW_WINDOW:
                        handleWindowVisibility((IBinder)msg.obj, true);
                        break;
                    case HIDE_WINDOW:
                        handleWindowVisibility((IBinder)msg.obj, false);
                        break;
                    case RESUME_ACTIVITY:
                        handleResumeActivity((IBinder)msg.obj, true,
                                msg.arg1 != 0);
                        break;
                    case SEND_RESULT:
                        handleSendResult((ResultData)msg.obj);
                        break;
                    case DESTROY_ACTIVITY:
                        handleDestroyActivity((IBinder)msg.obj, msg.arg1 != 0,
                                msg.arg2, false);
                        break;
                    case BIND_APPLICATION:
                        AppBindData data = (AppBindData)msg.obj;
                        handleBindApplication(data);
                        break;
                    case EXIT_APPLICATION:
                        if (mInitialApplication != null) {
                            mInitialApplication.onTerminate();
                        }
                        Looper.myLooper().quit();
                        break;
                    case NEW_INTENT:
                        handleNewIntent((NewIntentData)msg.obj);
                        break;
                    case RECEIVER:
                        handleReceiver((ReceiverData)msg.obj);
                        maybeSnapshot();
                        break;
                    case CREATE_SERVICE:
                        handleCreateService((CreateServiceData)msg.obj);
                        break;
                    case BIND_SERVICE:
                        handleBindService((BindServiceData)msg.obj);
                        break;
                    case UNBIND_SERVICE:
                        handleUnbindService((BindServiceData)msg.obj);
                        break;
                    case SERVICE_ARGS:
                        handleServiceArgs((ServiceArgsData)msg.obj);
                        break;
                    case STOP_SERVICE:
                        handleStopService((IBinder)msg.obj);
                        maybeSnapshot();
                        break;
                    case REQUEST_THUMBNAIL:
                        handleRequestThumbnail((IBinder)msg.obj);
                        break;
                    case CONFIGURATION_CHANGED:
                        handleConfigurationChanged((Configuration)msg.obj);
                        break;
                    case CLEAN_UP_CONTEXT:
                        ContextCleanupInfo cci = (ContextCleanupInfo)msg.obj;
                        cci.context.performFinalCleanup(cci.who, cci.what);
                        break;
                    case GC_WHEN_IDLE:
                        scheduleGcIdler();
                        break;
                    case DUMP_SERVICE:
                        handleDumpService((DumpServiceInfo)msg.obj);
                        break;
                    case LOW_MEMORY:
                        handleLowMemory();
                        break;
                    case ACTIVITY_CONFIGURATION_CHANGED:
                        handleActivityConfigurationChanged((IBinder)msg.obj);
                        break;
                    case PROFILER_CONTROL:
                        handleProfilerControl(msg.arg1 != 0, (ProfilerControlData)msg.obj);
                        break;
                    case CREATE_BACKUP_AGENT:
                        handleCreateBackupAgent((CreateBackupAgentData)msg.obj);
                        break;
                    case DESTROY_BACKUP_AGENT:
                        handleDestroyBackupAgent((CreateBackupAgentData)msg.obj);
                        break;
                    case SUICIDE:
                        Process.killProcess(Process.myPid());
                        break;
                    case REMOVE_PROVIDER:
                        completeRemoveProvider((IContentProvider)msg.obj);
                        break;
                    case ENABLE_JIT:
                        ensureJitEnabled();
                        break;
                    case DISPATCH_PACKAGE_BROADCAST:
                        handleDispatchPackageBroadcast(msg.arg1, (String[])msg.obj);
                        break;
                    case SCHEDULE_CRASH:
                        throw new RemoteServiceException((String)msg.obj);
                }
                if (DEBUG_MESSAGES) Slog.v(TAG, "<<< done: " + msg.what);
            }
    
}

```

可以看出H的handleMessage()方法处理了很多消息，其中就有PAUSE_ACTIVITY消息。

### 16 ActivityThread.handlePauseActivity(IBinder token, boolean finished, boolean userLeaving, int configChanges) 

```java
public final class ActivityThread {

    final HashMap<IBinder, ActivityClientRecord> mActivities
            = new HashMap<IBinder, ActivityClientRecord>();

    private final void handlePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges) {
        ActivityClientRecord r = mActivities.get(token);
        if (r != null) {
        
            //向Launcher组件发送一个用户离开事件的通知
            //Slog.v(TAG, "userLeaving=" + userLeaving + " handling pause of " + r);
            if (userLeaving) {
                performUserLeavingActivity(r);
            }

            r.activity.mConfigChangeFlags |= configChanges;
            //向Launcher组件发送一个终止事件的通知
            Bundle state = performPauseActivity(token, finished, true);

             //等待Launcher组件完成所有的写入操作，以便Launcher组件重新进入onResumed状态时
             //可以恢复到之前的状态
            // Make sure any pending writes are now committed.
            QueuedWork.waitToFinish();
            
            // Tell the activity manager we have paused.
            try {
                ActivityManagerNative.getDefault().activityPaused(token, state);
            } catch (RemoteException ex) {
            }
        }
    }

}
```

不同的组件中用不同的类来描述Activity对象，我们先来看看它们的对象关系：

```
在ActivityThread中：Activity组件->ActivityClientRecord
在ActivityManagerService中：Activity组件->ActivityRecord
```

描述Activity组件的ActivityClientRecord对象都保存在mActivities中，ActivityClientRecord r = mActivities.get(token)是为了在mActivities找打一个
用来描述Launcher组件的ActivityClientRecord对象。

我们看看这个函数做了哪些事情：

```
1 调用performUserLeavingActivity(r)向Launcher组件发送一个用户离开事件的通知
2 调用performPauseActivity(token, finished, true)向Launcher组件发送一个终止事件的通知
3 调用QueuedWork.waitToFinish()等待Launcher组件完成所有的写入操作，以便Launcher组件重新进入onResumed状态时可以恢复到之前的状态。
4 ActivityManagerNative.getDefault()获取ActivityManagerService的一个代理对象，然后再调用activityPaused(token, state)通知ActivityManagerService
，Launcher组件已经进入Paused状态了。
```

自此，Launcher组件正式进入Paused状态，可算把你暂停了。<img src="https://github.com/guoxiaoxing/emoji/raw/master/emoji/d_erha.png" width="30" height="30" align="bottom"/>

这个时候就可把我们的目标Activity组件启动起来了。

### 17 ActivityManagerProxy.activityPaused(IBinder token, Bundle state)

```java

class ActivityManagerProxy implements IActivityManager{

    public void activityPaused(IBinder token, Bundle state) throws RemoteException
    {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(token);
        data.writeBundle(state);
        mRemote.transact(ACTIVITY_PAUSED_TRANSACTION, data, reply, 0);
        reply.readException();
        data.recycle();
        reply.recycle();
    }
}
```

将handlePauseActivity()传递过来的参数包装成一个Parcel对象，并发起一个ACTIVITY_PAUSED_TRANSACTION进程通信请求。


### 18 ActivityManagerService.activityPaused(IBinder token, Bundle icicle)

```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback{

    public final void activityPaused(IBinder token, Bundle icicle) {
        // Refuse possible leaked file descriptors
        if (icicle != null && icicle.hasFileDescriptors()) {
            throw new IllegalArgumentException("File descriptors passed in Bundle");
        }

        final long origId = Binder.clearCallingIdentity();
        mMainStack.activityPaused(token, icicle, false);
        Binder.restoreCallingIdentity(origId);
    }

}
```

IBinder token还是跟前面一样指向ActivityManagerService中与Launcher组件对应的一个ActivityRecord对象。该函数会调用ActivityStack.activityPaused()方法
进一步就处理ACTIVITY_PAUSED_TRANSACTION进程通信请求。

### 19 ActivityStack.activityPaused(IBinder token, Bundle icicle, boolean timeout)

```java
public class ActivityStack {

    final ActivityManagerService mService;

    final void activityPaused(IBinder token, Bundle icicle, boolean timeout) {
        if (DEBUG_PAUSE) Slog.v(
            TAG, "Activity paused: token=" + token + ", icicle=" + icicle
            + ", timeout=" + timeout);

        ActivityRecord r = null;

        synchronized (mService) {
            int index = indexOfTokenLocked(token);
            if (index >= 0) {
                //根据token查找mHistory栈中对应的ActivityRecord对象
                r = (ActivityRecord)mHistory.get(index);
                if (!timeout) {
                    r.icicle = icicle;
                    r.haveState = true;
                }
                //移除PAUSE_TIMEOUT_MSG消息，因为Launcher组件已经在规定的时间内完成ActivityManagerService给
                //它发送的终止通知了。
                mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
                if (mPausingActivity == r) {
                    //此时r与mPausingActivity指向的都是Launcher对应的ActivityRecord对象。把Launcher置为
                    //ActivityState.PAUSED状态。
                    r.state = ActivityState.PAUSED;
                    //执行目标Activity的启动操作
                    completePauseLocked();
                } else {
                    EventLog.writeEvent(EventLogTags.AM_FAILED_TO_PAUSE,
                            System.identityHashCode(r), r.shortComponentName,
                            mPausingActivity != null
                                ? mPausingActivity.shortComponentName : "(none)");
                }
            }
        }
    }

}
```

该函数执行了以下操作：

```
1 根据token查找mHistory栈中对应的ActivityRecord对象。
2 移除PAUSE_TIMEOUT_MSG消息，因为Launcher组件已经在规定的时间内完成ActivityManagerService给它发送的终止通知了。
3 把Launcher置为ActivityState.PAUSED状态。
4 调用completePauseLocked()，执行目标Activity的启动操作。
```

### 20 ActivityStack.completePauseLocked()

```java
public class ActivityStack{

     private final void completePauseLocked() {
            ActivityRecord prev = mPausingActivity;
            if (DEBUG_PAUSE) Slog.v(TAG, "Complete pause: " + prev);

            if (prev != null) {
                if (prev.finishing) {
                    if (DEBUG_PAUSE) Slog.v(TAG, "Executing finish of activity: " + prev);
                    prev = finishCurrentActivityLocked(prev, FINISH_AFTER_VISIBLE);
                } else if (prev.app != null) {
                    if (DEBUG_PAUSE) Slog.v(TAG, "Enqueueing pending stop: " + prev);
                    if (prev.waitingVisible) {
                        prev.waitingVisible = false;
                        mWaitingVisibleActivities.remove(prev);
                        if (DEBUG_SWITCH || DEBUG_PAUSE) Slog.v(
                                TAG, "Complete pause, no longer waiting: " + prev);
                    }
                    if (prev.configDestroy) {
                        // The previous is being paused because the configuration
                        // is changing, which means it is actually stopping...
                        // To juggle the fact that we are also starting a new
                        // instance right now, we need to first completely stop
                        // the current instance before starting the new one.
                        if (DEBUG_PAUSE) Slog.v(TAG, "Destroying after pause: " + prev);
                        destroyActivityLocked(prev, true);
                    } else {
                        mStoppingActivities.add(prev);
                        if (mStoppingActivities.size() > 3) {
                            // If we already have a few activities waiting to stop,
                            // then give up on things going idle and start clearing
                            // them out.
                            if (DEBUG_PAUSE) Slog.v(TAG, "To many pending stops, forcing idle");
                            Message msg = Message.obtain();
                            msg.what = IDLE_NOW_MSG;
                            mHandler.sendMessage(msg);
                        }
                    }
                } else {
                    if (DEBUG_PAUSE) Slog.v(TAG, "App died during pause, not stopping: " + prev);
                    prev = null;
                }
                //mPausingActivity置为null，表示当前正在终止的Activity组件已经进入Paused状态了。
                mPausingActivity = null;
            }

            if (!mService.mSleeping && !mService.mShuttingDown) {
                //如果系统没有处于睡眠或关闭状态，则启动位于栈顶的Activity组件。
                resumeTopActivityLocked(prev);
            } else {
                if (mGoingToSleep.isHeld()) {
                    mGoingToSleep.release();
                }
                if (mService.mShuttingDown) {
                    mService.notifyAll();
                }
            }

            if (prev != null) {
                prev.resumeKeyDispatchingLocked();
            }

            if (prev.app != null && prev.cpuTimeAtResume > 0
                    && mService.mBatteryStatsService.isOnBattery()) {
                long diff = 0;
                synchronized (mService.mProcessStatsThread) {
                    diff = mService.mProcessStats.getCpuTimeForPid(prev.app.pid)
                            - prev.cpuTimeAtResume;
                }
                if (diff > 0) {
                    BatteryStatsImpl bsi = mService.mBatteryStatsService.getActiveStatistics();
                    synchronized (bsi) {
                        BatteryStatsImpl.Uid.Proc ps =
                                bsi.getProcessStatsLocked(prev.info.applicationInfo.uid,
                                prev.info.packageName);
                        if (ps != null) {
                            ps.addForegroundTimeLocked(diff);
                        }
                    }
                }
            }
            prev.cpuTimeAtResume = 0; // reset it
        }
}

```
mPausingActivity置为null，表示当前正在终止的Activity组件已经进入Paused状态了，调用resumeTopActivityLocked(prev)启动位于栈顶的Activity组件。

### 21 ActivityStack.resumeTopActivityLocked(ActivityRecord prev)

```java
public class ActivityStack{

      /**
         * Ensure that the top activity in the stack is resumed.
         *
         * @param prev The previously resumed activity, for when in the process
         * of pausing; can be null to call from elsewhere.
         *
         * @return Returns true if something is being resumed, or false if
         * nothing happened.
         */
        final boolean resumeTopActivityLocked(ActivityRecord prev) {

            //栈顶Activity，它指向即将启动的目标Activity组件
            // Find the first activity that is not finishing.
            ActivityRecord next = topRunningActivityLocked(null);

            // Remember how we'll process this pause/resume situation, and ensure
            // that the state is reset however we wind up proceeding.
            final boolean userLeaving = mUserLeaving;
            mUserLeaving = false;

            if (next == null) {
                // There are no more activities!  Let's just start up the
                // Launcher...
                if (mMainStack) {
                    return mService.startHomeActivityLocked();
                }
            }

            next.delayedResume = false;

            // If the top activity is the resumed one, nothing to do.
            if (mResumedActivity == next && next.state == ActivityState.RESUMED) {
                // Make sure we have executed any pending transitions, since there
                // should be nothing left to do at this point.
                mService.mWindowManager.executeAppTransition();
                mNoAnimActivities.clear();
                return false;
            }

            // If we are sleeping, and there is no resumed activity, and the top
            // activity is paused, well that is the state we want.
            if ((mService.mSleeping || mService.mShuttingDown)
                    && mLastPausedActivity == next && next.state == ActivityState.PAUSED) {
                // Make sure we have executed any pending transitions, since there
                // should be nothing left to do at this point.
                mService.mWindowManager.executeAppTransition();
                mNoAnimActivities.clear();
                return false;
            }

            // The activity may be waiting for stop, but that is no longer
            // appropriate for it.
            mStoppingActivities.remove(next);
            mWaitingVisibleActivities.remove(next);

            if (DEBUG_SWITCH) Slog.v(TAG, "Resuming " + next);

            //上一步中已经将mPausingActivity置为null
            // If we are currently pausing an activity, then don't do anything
            // until that is done.
            if (mPausingActivity != null) {
                if (DEBUG_SWITCH) Slog.v(TAG, "Skip resume: pausing=" + mPausingActivity);
                return false;
            }

            // Okay we are now going to start a switch, to 'next'.  We may first
            // have to pause the current activity, but this is an important point
            // where we have decided to go to 'next' so keep track of that.
            // XXX "App Redirected" dialog is getting too many false positives
            // at this point, so turn off for now.
            if (false) {
                if (mLastStartedActivity != null && !mLastStartedActivity.finishing) {
                    long now = SystemClock.uptimeMillis();
                    final boolean inTime = mLastStartedActivity.startTime != 0
                            && (mLastStartedActivity.startTime + START_WARN_TIME) >= now;
                    final int lastUid = mLastStartedActivity.info.applicationInfo.uid;
                    final int nextUid = next.info.applicationInfo.uid;
                    if (inTime && lastUid != nextUid
                            && lastUid != next.launchedFromUid
                            && mService.checkPermission(
                                    android.Manifest.permission.STOP_APP_SWITCHES,
                                    -1, next.launchedFromUid)
                            != PackageManager.PERMISSION_GRANTED) {
                        mService.showLaunchWarningLocked(mLastStartedActivity, next);
                    } else {
                        next.startTime = now;
                        mLastStartedActivity = next;
                    }
                } else {
                    next.startTime = SystemClock.uptimeMillis();
                    mLastStartedActivity = next;
                }
            }

            // We need to start pausing the current activity so the top one
            // can be resumed...
            if (mResumedActivity != null) {
                if (DEBUG_SWITCH) Slog.v(TAG, "Skip resume: need to start pausing");
                startPausingLocked(userLeaving, false);
                return true;
            }

            if (prev != null && prev != next) {
                if (!prev.waitingVisible && next != null && !next.nowVisible) {
                    prev.waitingVisible = true;
                    mWaitingVisibleActivities.add(prev);
                    if (DEBUG_SWITCH) Slog.v(
                            TAG, "Resuming top, waiting visible to hide: " + prev);
                } else {
                    // The next activity is already visible, so hide the previous
                    // activity's windows right now so we can show the new one ASAP.
                    // We only do this if the previous is finishing, which should mean
                    // it is on top of the one being resumed so hiding it quickly
                    // is good.  Otherwise, we want to do the normal route of allowing
                    // the resumed activity to be shown so we can decide if the
                    // previous should actually be hidden depending on whether the
                    // new one is found to be full-screen or not.
                    if (prev.finishing) {
                        mService.mWindowManager.setAppVisibility(prev, false);
                        if (DEBUG_SWITCH) Slog.v(TAG, "Not waiting for visible to hide: "
                                + prev + ", waitingVisible="
                                + (prev != null ? prev.waitingVisible : null)
                                + ", nowVisible=" + next.nowVisible);
                    } else {
                        if (DEBUG_SWITCH) Slog.v(TAG, "Previous already visible but still waiting to hide: "
                            + prev + ", waitingVisible="
                            + (prev != null ? prev.waitingVisible : null)
                            + ", nowVisible=" + next.nowVisible);
                    }
                }
            }

            // We are starting up the next activity, so tell the window manager
            // that the previous one will be hidden soon.  This way it can know
            // to ignore it when computing the desired screen orientation.
            if (prev != null) {
                if (prev.finishing) {
                    if (DEBUG_TRANSITION) Slog.v(TAG,
                            "Prepare close transition: prev=" + prev);
                    if (mNoAnimActivities.contains(prev)) {
                        mService.mWindowManager.prepareAppTransition(WindowManagerPolicy.TRANSIT_NONE);
                    } else {
                        mService.mWindowManager.prepareAppTransition(prev.task == next.task
                                ? WindowManagerPolicy.TRANSIT_ACTIVITY_CLOSE
                                : WindowManagerPolicy.TRANSIT_TASK_CLOSE);
                    }
                    mService.mWindowManager.setAppWillBeHidden(prev);
                    mService.mWindowManager.setAppVisibility(prev, false);
                } else {
                    if (DEBUG_TRANSITION) Slog.v(TAG,
                            "Prepare open transition: prev=" + prev);
                    if (mNoAnimActivities.contains(next)) {
                        mService.mWindowManager.prepareAppTransition(WindowManagerPolicy.TRANSIT_NONE);
                    } else {
                        mService.mWindowManager.prepareAppTransition(prev.task == next.task
                                ? WindowManagerPolicy.TRANSIT_ACTIVITY_OPEN
                                : WindowManagerPolicy.TRANSIT_TASK_OPEN);
                    }
                }
                if (false) {
                    mService.mWindowManager.setAppWillBeHidden(prev);
                    mService.mWindowManager.setAppVisibility(prev, false);
                }
            } else if (mHistory.size() > 1) {
                if (DEBUG_TRANSITION) Slog.v(TAG,
                        "Prepare open transition: no previous");
                if (mNoAnimActivities.contains(next)) {
                    mService.mWindowManager.prepareAppTransition(WindowManagerPolicy.TRANSIT_NONE);
                } else {
                    mService.mWindowManager.prepareAppTransition(WindowManagerPolicy.TRANSIT_ACTIVITY_OPEN);
                }
            }

            if (next.app != null && next.app.thread != null) {
                if (DEBUG_SWITCH) Slog.v(TAG, "Resume running: " + next);

                // This activity is now becoming visible.
                mService.mWindowManager.setAppVisibility(next, true);

                ActivityRecord lastResumedActivity = mResumedActivity;
                ActivityState lastState = next.state;

                mService.updateCpuStats();

                next.state = ActivityState.RESUMED;
                mResumedActivity = next;
                next.task.touchActiveTime();
                mService.updateLruProcessLocked(next.app, true, true);
                updateLRUListLocked(next);

                // Have the window manager re-evaluate the orientation of
                // the screen based on the new activity order.
                boolean updated = false;
                if (mMainStack) {
                    synchronized (mService) {
                        Configuration config = mService.mWindowManager.updateOrientationFromAppTokens(
                                mService.mConfiguration,
                                next.mayFreezeScreenLocked(next.app) ? next : null);
                        if (config != null) {
                            next.frozenBeforeDestroy = true;
                        }
                        updated = mService.updateConfigurationLocked(config, next);
                    }
                }
                if (!updated) {
                    // The configuration update wasn't able to keep the existing
                    // instance of the activity, and instead started a new one.
                    // We should be all done, but let's just make sure our activity
                    // is still at the top and schedule another run if something
                    // weird happened.
                    ActivityRecord nextNext = topRunningActivityLocked(null);
                    if (DEBUG_SWITCH) Slog.i(TAG,
                            "Activity config changed during resume: " + next
                            + ", new next: " + nextNext);
                    if (nextNext != next) {
                        // Do over!
                        mHandler.sendEmptyMessage(RESUME_TOP_ACTIVITY_MSG);
                    }
                    if (mMainStack) {
                        mService.setFocusedActivityLocked(next);
                    }
                    ensureActivitiesVisibleLocked(null, 0);
                    mService.mWindowManager.executeAppTransition();
                    mNoAnimActivities.clear();
                    return true;
                }

                try {
                    // Deliver all pending results.
                    ArrayList a = next.results;
                    if (a != null) {
                        final int N = a.size();
                        if (!next.finishing && N > 0) {
                            if (DEBUG_RESULTS) Slog.v(
                                    TAG, "Delivering results to " + next
                                    + ": " + a);
                            next.app.thread.scheduleSendResult(next, a);
                        }
                    }

                    if (next.newIntents != null) {
                        next.app.thread.scheduleNewIntent(next.newIntents, next);
                    }

                    EventLog.writeEvent(EventLogTags.AM_RESUME_ACTIVITY,
                            System.identityHashCode(next),
                            next.task.taskId, next.shortComponentName);

                    next.app.thread.scheduleResumeActivity(next,
                            mService.isNextTransitionForward());

                    pauseIfSleepingLocked();

                } catch (Exception e) {
                    // Whoops, need to restart this activity!
                    next.state = lastState;
                    mResumedActivity = lastResumedActivity;
                    Slog.i(TAG, "Restarting because process died: " + next);
                    if (!next.hasBeenLaunched) {
                        next.hasBeenLaunched = true;
                    } else {
                        if (SHOW_APP_STARTING_PREVIEW && mMainStack) {
                            mService.mWindowManager.setAppStartingWindow(
                                    next, next.packageName, next.theme,
                                    next.nonLocalizedLabel,
                                    next.labelRes, next.icon, null, true);
                        }
                    }
                    //执行目标Activity启动操作。
                    startSpecificActivityLocked(next, true, false);
                    return true;
                }

                // From this point on, if something goes wrong there is no way
                // to recover the activity.
                try {
                    next.visible = true;
                    completeResumeLocked(next);
                } catch (Exception e) {
                    // If any exception gets thrown, toss away this
                    // activity and try the next one.
                    Slog.w(TAG, "Exception thrown during resume of " + next, e);
                    requestFinishActivityLocked(next, Activity.RESULT_CANCELED, null,
                            "resume-exception");
                    return true;
                }

                // Didn't need to use the icicle, and it is now out of date.
                next.icicle = null;
                next.haveState = false;
                next.stopped = false;

            } else {
                // Whoops, need to restart this activity!
                if (!next.hasBeenLaunched) {
                    next.hasBeenLaunched = true;
                } else {
                    if (SHOW_APP_STARTING_PREVIEW) {
                        mService.mWindowManager.setAppStartingWindow(
                                next, next.packageName, next.theme,
                                next.nonLocalizedLabel,
                                next.labelRes, next.icon, null, true);
                    }
                    if (DEBUG_SWITCH) Slog.v(TAG, "Restarting: " + next);
                }
                startSpecificActivityLocked(next, true, true);
            }

            return true;
        }

}
```

在第10步中，ActivityManagerService已经尝试调用ActivityManagerService.resumeTopActivityLocked()来启动目标Activity组件了，但是
那个时候Launcher组件尚未进入Paused状态（即ActivityStack.mResumedActivity != null）所以先调用ActivityManagerService.startPausingLocked()
来终止Launcher组件。

接下来调用startSpecificActivityLocked(next, true, false)，执行目标Activity启动操作。


### 22 ActivityStack.startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig)

```java
public class ActivityStack{

    private final void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid);

        if (r.launchTime == 0) {
            r.launchTime = SystemClock.uptimeMillis();
            if (mInitialStartTime == 0) {
                mInitialStartTime = r.launchTime;
            }
        } else if (mInitialStartTime == 0) {
            mInitialStartTime = SystemClock.uptimeMillis();
        }

        if (app != null && app.thread != null) {
            try {
                //如果存在，则直接通知该应用进程将该Activity启动起来。
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }

        //如果不存在，则用该Activity的用户ID和进程名称创建一个新的应用进程，再由该进程将该Activity启动起来。
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false);
    }

}
```

在ActivityManagerService中，每个Activity组件都有一个用户ID和一个进程名称。

```
用户ID：安装该Activity时由PackageManagerService分配的。
进程名称：由Activity组件的android:process属性所决定的。
```
ActivityManagerService在启动Activity时会用它的用户ID和进程名称来检查系统中是否存在一个对应的应用进程。

如果存在，则直接通知该应用进程将该Activity启动起来。
如果不存在，则用该Activity的用户ID和进程名称创建一个新的应用进程，再由该进程将该Activity启动起来。

我们这里的目标Activity是第一次启动，所以它会调用startProcessLocked()为目标Activity创建一个新的进程。

### 23 ActivityManagerService.startProcessLocked(String processName, ApplicationInfo info, boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName, boolean allowWhileBooting)

```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback{

    final ProcessRecord startProcessLocked(String processName,
            ApplicationInfo info, boolean knownToBeDead, int intentFlags,
            String hostingType, ComponentName hostingName, boolean allowWhileBooting) {

        //检查请求创建的应用进程是否已经存在，如果已经存在，则不再创建。
        ProcessRecord app = getProcessRecordLocked(processName, info.uid);
        // We don't have to do anything more if:
        // (1) There is an existing application record; and
        // (2) The caller doesn't think it is dead, OR there is no thread
        //     object attached to it so we know it couldn't have crashed; and
        // (3) There is a pid assigned to it, so it is either starting or
        //     already running.
        if (DEBUG_PROCESSES) Slog.v(TAG, "startProcess: text=" + processName
                + " app=" + app + " knownToBeDead=" + knownToBeDead
                + " thread=" + (app != null ? app.thread : null)
                + " pid=" + (app != null ? app.pid : -1));
        if (app != null && app.pid > 0) {
            if (!knownToBeDead || app.thread == null) {
                // We already have the app running, or are waiting for it to
                // come up (we have a pid but not yet its thread), so keep it.
                if (DEBUG_PROCESSES) Slog.v(TAG, "App already running: " + app);
                return app;
            } else {
                // An application record is attached to a previous process,
                // clean it up now.
                if (DEBUG_PROCESSES) Slog.v(TAG, "App died: " + app);
                handleAppDiedLocked(app, true);
            }
        }

        String hostingNameStr = hostingName != null
                ? hostingName.flattenToShortString() : null;

        if ((intentFlags&Intent.FLAG_FROM_BACKGROUND) != 0) {
            // If we are in the background, then check to see if this process
            // is bad.  If so, we will just silently fail.
            if (mBadProcesses.get(info.processName, info.uid) != null) {
                if (DEBUG_PROCESSES) Slog.v(TAG, "Bad process: " + info.uid
                        + "/" + info.processName);
                return null;
            }
        } else {
            // When the user is explicitly starting a process, then clear its
            // crash count so that we won't make it bad until they see at
            // least one crash dialog again, and make the process good again
            // if it had been bad.
            if (DEBUG_PROCESSES) Slog.v(TAG, "Clearing bad process: " + info.uid
                    + "/" + info.processName);
            mProcessCrashTimes.remove(info.processName, info.uid);
            if (mBadProcesses.get(info.processName, info.uid) != null) {
                EventLog.writeEvent(EventLogTags.AM_PROC_GOOD, info.uid,
                        info.processName);
                mBadProcesses.remove(info.processName, info.uid);
                if (app != null) {
                    app.bad = false;
                }
            }
        }

        if (app == null) {
            app = newProcessRecordLocked(null, info, processName);
            mProcessNames.put(processName, info.uid, app);
        } else {
            // If this is a new package in the process, add the package to the list
            app.addPackage(info.packageName);
        }

        // If the system is not ready yet, then hold off on starting this
        // process until it is.
        if (!mProcessesReady
                && !isAllowedWhileBooting(info)
                && !allowWhileBooting) {
            if (!mProcessesOnHold.contains(app)) {
                mProcessesOnHold.add(app);
            }
            if (DEBUG_PROCESSES) Slog.v(TAG, "System not ready, putting on hold: " + app);
            return app;
        }
        //如果不存在，则调用它的重载函数创建一个新的应用进程。
        startProcessLocked(app, hostingType, hostingNameStr);
        return (app.pid != 0) ? app : null;
    }

}

```

该函数会检查请求创建的应用进程是否已经存在，如果已经存在，则不再创建。如果不存在，则调用它的重载函数startProcessLocked(app, hostingType, hostingNameStr)创建一个新的应用进程。
具体实现：

```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback{

      private final void startProcessLocked(ProcessRecord app,
                String hostingType, String hostingNameStr) {
            if (app.pid > 0 && app.pid != MY_PID) {
                synchronized (mPidsSelfLocked) {
                    mPidsSelfLocked.remove(app.pid);
                    mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
                }
                app.pid = 0;
            }

            if (DEBUG_PROCESSES && mProcessesOnHold.contains(app)) Slog.v(TAG,
                    "startProcessLocked removing on hold: " + app);
            mProcessesOnHold.remove(app);

            updateCpuStats();

            System.arraycopy(mProcDeaths, 0, mProcDeaths, 1, mProcDeaths.length-1);
            mProcDeaths[0] = 0;

            try {
                //获取将要创建的应用进程的用户ID与用户组ID。
                int uid = app.info.uid;
                int[] gids = null;
                try {
                    gids = mContext.getPackageManager().getPackageGids(
                            app.info.packageName);
                } catch (PackageManager.NameNotFoundException e) {
                    Slog.w(TAG, "Unable to retrieve gids", e);
                }
                if (mFactoryTest != SystemServer.FACTORY_TEST_OFF) {
                    if (mFactoryTest == SystemServer.FACTORY_TEST_LOW_LEVEL
                            && mTopComponent != null
                            && app.processName.equals(mTopComponent.getPackageName())) {
                        uid = 0;
                    }
                    if (mFactoryTest == SystemServer.FACTORY_TEST_HIGH_LEVEL
                            && (app.info.flags&ApplicationInfo.FLAG_FACTORY_TEST) != 0) {
                        uid = 0;
                    }
                }
                int debugFlags = 0;
                if ((app.info.flags & ApplicationInfo.FLAG_DEBUGGABLE) != 0) {
                    debugFlags |= Zygote.DEBUG_ENABLE_DEBUGGER;
                }
                // Run the app in safe mode if its manifest requests so or the
                // system is booted in safe mode.
                if ((app.info.flags & ApplicationInfo.FLAG_VM_SAFE_MODE) != 0 ||
                    Zygote.systemInSafeMode == true) {
                    debugFlags |= Zygote.DEBUG_ENABLE_SAFEMODE;
                }
                if ("1".equals(SystemProperties.get("debug.checkjni"))) {
                    debugFlags |= Zygote.DEBUG_ENABLE_CHECKJNI;
                }
                if ("1".equals(SystemProperties.get("debug.assert"))) {
                    debugFlags |= Zygote.DEBUG_ENABLE_ASSERT;
                }
                //创建一个新的应用进程。pid是一个大于0的ID
                int pid = Process.start("android.app.ActivityThread",
                        mSimpleProcessManagement ? app.processName : null, uid, uid,
                        gids, debugFlags, null);
                BatteryStatsImpl bs = app.batteryStats.getBatteryStats();
                synchronized (bs) {
                    if (bs.isOnBattery()) {
                        app.batteryStats.incStartsLocked();
                    }
                }

                EventLog.writeEvent(EventLogTags.AM_PROC_START, pid, uid,
                        app.processName, hostingType,
                        hostingNameStr != null ? hostingNameStr : "");

                if (app.persistent) {
                    Watchdog.getInstance().processStarted(app.processName, pid);
                }

                StringBuilder buf = mStringBuilder;
                buf.setLength(0);
                buf.append("Start proc ");
                buf.append(app.processName);
                buf.append(" for ");
                buf.append(hostingType);
                if (hostingNameStr != null) {
                    buf.append(" ");
                    buf.append(hostingNameStr);
                }
                buf.append(": pid=");
                buf.append(pid);
                buf.append(" uid=");
                buf.append(uid);
                buf.append(" gids={");
                if (gids != null) {
                    for (int gi=0; gi<gids.length; gi++) {
                        if (gi != 0) buf.append(", ");
                        buf.append(gids[gi]);

                    }
                }
                buf.append("}");
                Slog.i(TAG, buf.toString());
                if (pid == 0 || pid == MY_PID) {
                    // Processes are being emulated with threads.
                    app.pid = MY_PID;
                    app.removed = false;
                    mStartingProcesses.add(app);
                } else if (pid > 0) {
                    app.pid = pid;
                    app.removed = false;
                    synchronized (mPidsSelfLocked) {
                        //以pid为关键字将app指向的一个ProcessRecord对象保存在mPidsSelfLocked中
                        this.mPidsSelfLocked.put(pid, app);
                        //向ActivityManagerService所在线程发送一个PROC_START_TIMEOUT_MSG消息，
                        Message msg = mHandler.obtainMessage(PROC_START_TIMEOUT_MSG);
                        msg.obj = app;
                        //该消息要求PROC_START_TIMEOUT时间内被处理，也就是说新的进必须在PROC_START_TIMEOUT
                        //时间内完成启动，并向ActivityManagerService发送一个启动通知，以便ActivityManagerService
                        //去启动Activity组件，否则ActivityManagerService则任务进程创建超时，则无法启动Activity。
                        mHandler.sendMessageDelayed(msg, PROC_START_TIMEOUT);
                    }
                } else {
                    app.pid = 0;
                    RuntimeException e = new RuntimeException(
                            "Failure starting process " + app.processName
                            + ": returned pid=" + pid);
                    Slog.e(TAG, e.getMessage(), e);
                }
            } catch (RuntimeException e) {
                // XXX do better error recovery.
                app.pid = 0;
                Slog.e(TAG, "Failure starting process " + app.processName, e);
            }
        }

}
```

这个重载函数做了以下事情：

```
1 获取将要创建的应用进程的用户ID与用户组ID。
2 向ActivityManagerService所在线程发送一个PROC_START_TIMEOUT_MSG消息，该消息要求PROC_START_TIMEOUT时间内被处理，也就是说新的进必须在PROC_START_TIMEOUT,
时间内完成启动，并向ActivityManagerService发送一个启动通知，以便ActivityManagerService去启动Activity组件，否则ActivityManagerService则任务进程创建超时，则
无法启动Activity。
```

上述函数调用Process.start()来启动一个新的应用进程，指定该进程的进入函数为android.app.ActivityThread的静态成员函数main.

```
int pid = Process.start("android.app.ActivityThread",
            mSimpleProcessManagement ? app.processName : null, uid, uid,
            gids, debugFlags, null);
```

好了，我们接着看看进程是怎么创建的吧。

### 24 ActivityThread.main(String[] args)

```java
public final class ActivityThread {

    public static final void main(String[] args) {
        SamplingProfilerIntegration.start();

        Process.setArgV0("<pre-initialized>");

        //创建消息循环，并使当前进程进入该消息循环。
        Looper.prepareMainLooper();
        if (sMainThreadHandler == null) {
            sMainThreadHandler = new Handler();
        }

        //创建ActivityThread对象
        ActivityThread thread = new ActivityThread();
        //向ActivityManagerService发送一个启动完成通知
        thread.attach(false);

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Looper.loop();

        if (Process.supportsProcesses()) {
            throw new RuntimeException("Main thread loop unexpectedly exited");
        }

        thread.detach();
        String text = (thread.mInitialApplication != null)
            ? thread.mInitialApplication.getPackageName()
            : "<unknown>";
        Slog.i(TAG, "Main thread of " + text + " is now exiting");
    }

}
```

新的应用进程启动时主要做两件事情：

1 在进程中创建ActivityThread对象，创建ActivityThread对象时，会同时在其内部创建一个ApplicationThread对象，该对象是一个Binder本地对象，ActivityManagerService就是
通过它和应用进程通信的。

2 调用thread.attach(false)向ctivityManagerService发送启动完成通知。

3 调用Looper.prepareMainLooper()创建一个消息循环，并使当前进程进入到该消息循环中，

thread.attach(false)最终会去调用ActivityManagerProxy.attachApplication(IApplicationThread app)发送一个进程间通信请求。


### 25 ActivityManagerProxy.attachApplication(IApplicationThread app)

```java
public final class ActivityManagerProxy{

   public void attachApplication(IApplicationThread app) throws RemoteException{
       Parcel data = Parcel.obtain();
       Parcel reply = Parcel.obtain();
       data.writeInterfaceToken(IActivityManager.descriptor);
       data.writeStrongBinder(app.asBinder());
       mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
       reply.readException();
       data.recycle();
       reply.recycle();
   }
}
```

将传递过来的参数包装成一个Parcel对象，并发起一个ATTACH_APPLICATION_TRANSACTION进程通信请求。


### 26 ActivityManagerService.attachApplication(IApplicationThread thread)

```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
}
```
attachApplication(IApplicationThread thread)接收到了新应用进程发送过来的ATTACH_APPLICATION_TRANSACTION进程通信请求之后，它就知道
新的应用进程已经创建，接着调用ActivityManagerService.attachApplicationLocked(IApplicationThread thread, int pid)继续执行Activity
的启动操作。

### 27 ActivityManagerService.attachApplicationLocked(IApplicationThread thread, int pid)

```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

 private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

        // Find the application record that is being attached...  either via
        // the pid if we are running in multiple processes, or just pull the
        // next app record if we are emulating process with anonymous threads.
        ProcessRecord app;
        if (pid != MY_PID && pid >= 0) {
            synchronized (mPidsSelfLocked) {
                //通过新进程的pid取出对应的ProcessRecord对象，并保存在ProcessRecord app中。
                app = mPidsSelfLocked.get(pid);
            }
        } else if (mStartingProcesses.size() > 0) {
            app = mStartingProcesses.remove(0);
            app.setPid(pid);
        } else {
            app = null;
        }

        if (app == null) {
            Slog.w(TAG, "No pending application record for pid " + pid
                    + " (IApplicationThread " + thread + "); dropping process");
            EventLog.writeEvent(EventLogTags.AM_DROP_PROCESS, pid);
            if (pid > 0 && pid != MY_PID) {
                Process.killProcess(pid);
            } else {
                try {
                    thread.scheduleExit();
                } catch (Exception e) {
                    // Ignore exceptions.
                }
            }
            return false;
        }

        // If this application record is still attached to a previous
        // process, clean it up now.
        if (app.thread != null) {
            handleAppDiedLocked(app, true);
        }

        // Tell the process all about itself.

        if (localLOGV) Slog.v(
                TAG, "Binding process pid " + pid + " to record " + app);

        String processName = app.processName;
        try {
            thread.asBinder().linkToDeath(new AppDeathRecipient(
                    app, pid, thread), 0);
        } catch (RemoteException e) {
            app.resetPackageList();
            startProcessLocked(app, "link fail", processName);
            return false;
        }

        EventLog.writeEvent(EventLogTags.AM_PROC_BOUND, app.pid, app.processName);

        app.thread = thread;
        app.curAdj = app.setAdj = -100;
        app.curSchedGroup = Process.THREAD_GROUP_DEFAULT;
        app.setSchedGroup = Process.THREAD_GROUP_BG_NONINTERACTIVE;
        app.forcingToForeground = null;
        app.foregroundServices = false;
        app.debugging = false;

        //新进程已经在规定时间内创建，移除PROC_START_TIMEOUT_MSG消息
        mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);

        boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
        List providers = normalMode ? generateApplicationProvidersLocked(app) : null;

        if (!normalMode) {
            Slog.i(TAG, "Launching preboot mode app: " + app);
        }

        if (localLOGV) Slog.v(
            TAG, "New app record " + app
            + " thread=" + thread.asBinder() + " pid=" + pid);
        try {
            int testMode = IApplicationThread.DEBUG_OFF;
            if (mDebugApp != null && mDebugApp.equals(processName)) {
                testMode = mWaitForDebugger
                    ? IApplicationThread.DEBUG_WAIT
                    : IApplicationThread.DEBUG_ON;
                app.debugging = true;
                if (mDebugTransient) {
                    mDebugApp = mOrigDebugApp;
                    mWaitForDebugger = mOrigWaitForDebugger;
                }
            }

            // If the app is being launched for restore or full backup, set it up specially
            boolean isRestrictedBackupMode = false;
            if (mBackupTarget != null && mBackupAppName.equals(processName)) {
                isRestrictedBackupMode = (mBackupTarget.backupMode == BackupRecord.RESTORE)
                        || (mBackupTarget.backupMode == BackupRecord.BACKUP_FULL);
            }

            ensurePackageDexOpt(app.instrumentationInfo != null
                    ? app.instrumentationInfo.packageName
                    : app.info.packageName);
            if (app.instrumentationClass != null) {
                ensurePackageDexOpt(app.instrumentationClass.getPackageName());
            }
            if (DEBUG_CONFIGURATION) Slog.v(TAG, "Binding proc "
                    + processName + " with config " + mConfiguration);
            thread.bindApplication(processName, app.instrumentationInfo != null
                    ? app.instrumentationInfo : app.info, providers,
                    app.instrumentationClass, app.instrumentationProfileFile,
                    app.instrumentationArguments, app.instrumentationWatcher, testMode,
                    isRestrictedBackupMode || !normalMode,
                    mConfiguration, getCommonServicesLocked());
            updateLruProcessLocked(app, false, true);
            app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
        } catch (Exception e) {
            // todo: Yikes!  What should we do?  For now we will try to
            // start another process, but that could easily get us in
            // an infinite loop of restarting processes...
            Slog.w(TAG, "Exception thrown during bind!", e);

            app.resetPackageList();
            startProcessLocked(app, "bind fail", processName);
            return false;
        }

        // Remove this record from the list of starting applications.
        mPersistentStartingProcesses.remove(app);
        if (DEBUG_PROCESSES && mProcessesOnHold.contains(app)) Slog.v(TAG,
                "Attach application locked removing on hold: " + app);
        mProcessesOnHold.remove(app);

        boolean badApp = false;
        boolean didSomething = false;

        //查看栈顶Activity（也就是我们要启动的目标Activity）是否将要在当前进程中启动。
        // See if the top visible activity is waiting to run in this process...
        ActivityRecord hr = mMainStack.topRunningActivityLocked(null);
        if (hr != null && normalMode) {
            //检查目标Activity的用户ID与进程名称与app所描述的应用进程的用户ID与进程名称是否一致。
            if (hr.app == null && app.info.uid == hr.info.applicationInfo.uid
                    && processName.equals(hr.processName)) {
                try {
                    //如果一致，则继续执行目标Activity启动操作。
                    if (mMainStack.realStartActivityLocked(hr, app, true, true)) {
                        didSomething = true;
                    }
                } catch (Exception e) {
                    Slog.w(TAG, "Exception in new application when starting activity "
                          + hr.intent.getComponent().flattenToShortString(), e);
                    badApp = true;
                }
            } else {
                mMainStack.ensureActivitiesVisibleLocked(hr, null, processName, 0);
            }
        }

        // Find any services that should be running in this process...
        if (!badApp && mPendingServices.size() > 0) {
            ServiceRecord sr = null;
            try {
                for (int i=0; i<mPendingServices.size(); i++) {
                    sr = mPendingServices.get(i);
                    if (app.info.uid != sr.appInfo.uid
                            || !processName.equals(sr.processName)) {
                        continue;
                    }

                    mPendingServices.remove(i);
                    i--;
                    realStartServiceLocked(sr, app);
                    didSomething = true;
                }
            } catch (Exception e) {
                Slog.w(TAG, "Exception in new application when starting service "
                      + sr.shortName, e);
                badApp = true;
            }
        }

        // Check if the next broadcast receiver is in this process...
        BroadcastRecord br = mPendingBroadcast;
        if (!badApp && br != null && br.curApp == app) {
            try {
                mPendingBroadcast = null;
                processCurBroadcastLocked(br, app);
                didSomething = true;
            } catch (Exception e) {
                Slog.w(TAG, "Exception in new application when starting receiver "
                      + br.curComponent.flattenToShortString(), e);
                badApp = true;
                logBroadcastReceiverDiscardLocked(br);
                finishReceiverLocked(br.receiver, br.resultCode, br.resultData,
                        br.resultExtras, br.resultAbort, true);
                scheduleBroadcastsLocked();
                // We need to reset the state if we fails to start the receiver.
                br.state = BroadcastRecord.IDLE;
            }
        }

        // Check whether the next backup agent is in this process...
        if (!badApp && mBackupTarget != null && mBackupTarget.appInfo.uid == app.info.uid) {
            if (DEBUG_BACKUP) Slog.v(TAG, "New app is backup target, launching agent for " + app);
            ensurePackageDexOpt(mBackupTarget.appInfo.packageName);
            try {
                thread.scheduleCreateBackupAgent(mBackupTarget.appInfo, mBackupTarget.backupMode);
            } catch (Exception e) {
                Slog.w(TAG, "Exception scheduling backup agent creation: ");
                e.printStackTrace();
            }
        }

        if (badApp) {
            // todo: Also need to kill application to deal with all
            // kinds of exceptions.
            handleAppDiedLocked(app, false);
            return false;
        }

        if (!didSomething) {
            updateOomAdjLocked();
        }

        return true;
    }

}
```
该函数做了以下事情：

1 通过新进程的pid取出对应的ProcessRecord对象，并保存在ProcessRecord app中。
2 新进程已经在规定时间内创建，移除PROC_START_TIMEOUT_MSG消息.
3 查看栈顶Activity（也就是我们要启动的目标Activity）是否将要在当前进程中启动。这个主要检查目标Activity的用户ID与进程名称与app所描述的应用进程的用户ID与进程名称是否一致。

如果一致，则继续调用mMainStack.realStartActivityLocked(hr, app, true, true)，执行目标Activity启动操作。

### 28 ActivityStack.realStartActivityLocked(ActivityRecord r, ProcessRecord app, boolean andResume, boolean checkConfig)

```java
public class ActivityStack{

  final boolean realStartActivityLocked(ActivityRecord r,
            ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {

        r.startFreezingScreenLocked(app, 0);
        mService.mWindowManager.setAppVisibility(r, true);

        // Have the window manager re-evaluate the orientation of
        // the screen based on the new activity order.  Note that
        // as a result of this, it can call back into the activity
        // manager with a new orientation.  We don't care about that,
        // because the activity is not currently running so we are
        // just restarting it anyway.
        if (checkConfig) {
            Configuration config = mService.mWindowManager.updateOrientationFromAppTokens(
                    mService.mConfiguration,
                    r.mayFreezeScreenLocked(app) ? r : null);
            mService.updateConfigurationLocked(config, r);
        }

        //表示它描述的Activity组件是在参数app描述的应用进程中启动。
        r.app = app;

        if (localLOGV) Slog.v(TAG, "Launching: " + r);

        int idx = app.activities.indexOf(r);
        if (idx < 0) {
            //将目标Activity组件添加到该应用进程中。
            app.activities.add(r);
        }
        mService.updateLruProcessLocked(app, true, true);

        try {
            if (app.thread == null) {
                throw new RemoteException();
            }
            List<ResultInfo> results = null;
            List<Intent> newIntents = null;
            if (andResume) {
                results = r.results;
                newIntents = r.newIntents;
            }
            if (DEBUG_SWITCH) Slog.v(TAG, "Launching: " + r
                    + " icicle=" + r.icicle
                    + " with results=" + results + " newIntents=" + newIntents
                    + " andResume=" + andResume);
            if (andResume) {
                EventLog.writeEvent(EventLogTags.AM_RESTART_ACTIVITY,
                        System.identityHashCode(r),
                        r.task.taskId, r.shortComponentName);
            }
            if (r.isHomeActivity) {
                mService.mHomeProcess = app;
            }
            mService.ensurePackageDexOpt(r.intent.getComponent().getPackageName());

            //app.thread是一个类型为ApplicationTHreadProxy的Binder代理对象，向我们刚才新创建的应用进程发送一个
            //进程间通信请求，通知前面创建的应用进程启动目标Activity组件，该Activity组件由ActivityRecord r来描述。
            app.thread.scheduleLaunchActivity(new Intent(r.intent), r,
                    System.identityHashCode(r),
                    r.info, r.icicle, results, newIntents, !andResume,
                    mService.isNextTransitionForward());

            if ((app.info.flags&ApplicationInfo.FLAG_CANT_SAVE_STATE) != 0) {
                // This may be a heavy-weight process!  Note that the package
                // manager will ensure that only activity can run in the main
                // process of the .apk, which is the only thing that will be
                // considered heavy-weight.
                if (app.processName.equals(app.info.packageName)) {
                    if (mService.mHeavyWeightProcess != null
                            && mService.mHeavyWeightProcess != app) {
                        Log.w(TAG, "Starting new heavy weight process " + app
                                + " when already running "
                                + mService.mHeavyWeightProcess);
                    }
                    mService.mHeavyWeightProcess = app;
                    Message msg = mService.mHandler.obtainMessage(
                            ActivityManagerService.POST_HEAVY_NOTIFICATION_MSG);
                    msg.obj = r;
                    mService.mHandler.sendMessage(msg);
                }
            }

        } catch (RemoteException e) {
            if (r.launchFailed) {
                // This is the second time we failed -- finish activity
                // and give up.
                Slog.e(TAG, "Second failure launching "
                      + r.intent.getComponent().flattenToShortString()
                      + ", giving up", e);
                mService.appDiedLocked(app, app.pid, app.thread);
                requestFinishActivityLocked(r, Activity.RESULT_CANCELED, null,
                        "2nd-crash");
                return false;
            }

            // This is the first time we failed -- restart process and
            // retry.
            app.activities.remove(r);
            throw e;
        }

        r.launchFailed = false;
        if (updateLRUListLocked(r)) {
            Slog.w(TAG, "Activity " + r
                  + " being launched, but already in LRU list");
        }

        if (andResume) {
            // As part of the process of launching, ActivityThread also performs
            // a resume.
            r.state = ActivityState.RESUMED;
            r.icicle = null;
            r.haveState = false;
            r.stopped = false;
            mResumedActivity = r;
            r.task.touchActiveTime();
            completeResumeLocked(r);
            pauseIfSleepingLocked();
        } else {
            // This activity is not starting in the resumed state... which
            // should look like we asked it to pause+stop (but remain visible),
            // and it has done so and reported back the current icicle and
            // other state.
            r.state = ActivityState.STOPPED;
            r.stopped = true;
        }

        // Launch the new version setup screen if needed.  We do this -after-
        // launching the initial activity (that is, home), so that it can have
        // a chance to initialize itself while in the background, making the
        // switch back to it faster and look better.
        if (mMainStack) {
            mService.startSetupActivityLocked();
        }

        return true;
    }
}
```

该函数主要做了以下事情：

```
1 将目标Activity添加到新创建的进程的Activity组件列表中。
2 调用 app.thread.scheduleLaunchActivity()，向我们刚才新创建的应用进程发送一个进程间通信请求，通知前面创建的应用进程启动目标Activity组件。
```

这里说一下这个app.thread，它是一个类型为ApplicationTHreadProxy的Binder代理对象。

### 29 ApplicationThreadProxy.scheduleLaunchActivity(Intent intent, IBinder token, int ident, ActivityInfo info, Bundle state, List<ResultInfo> pendingResults, List<Intent> pendingNewIntents, boolean notResumed, boolean isForward)

```java
class ApplicationThreadProxy implements IApplicationThread{

    public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
            ActivityInfo info, Bundle state, List<ResultInfo> pendingResults,
    		List<Intent> pendingNewIntents, boolean notResumed, boolean isForward)
    		throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        intent.writeToParcel(data, 0);
        data.writeStrongBinder(token);
        data.writeInt(ident);
        info.writeToParcel(data, 0);
        data.writeBundle(state);
        data.writeTypedList(pendingResults);
        data.writeTypedList(pendingNewIntents);
        data.writeInt(notResumed ? 1 : 0);
        data.writeInt(isForward ? 1 : 0);
        mRemote.transact(SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
    
}
```

将传递过来的参数打包成一个Parcel对象，并发送一个类型为SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION进程间通信请求。

### 30 ActivityThread.scheduleRelaunchActivity(IBinder token, List<ResultInfo> pendingResults, List<Intent> pendingNewIntents, int configChanges, boolean notResumed, Configuration config)

```java
public final class ActivityThread{

    private final class ApplicationThread extends ApplicationThreadNative {
    
        // we use token to identify this activity without having to send the
        // activity itself back to the activity manager. (matters more with ipc)
        public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Bundle state, List<ResultInfo> pendingResults,
                List<Intent> pendingNewIntents, boolean notResumed, boolean isForward) {
            ActivityClientRecord r = new ActivityClientRecord();

            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.activityInfo = info;
            r.state = state;

            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;

            r.startsNotResumed = notResumed;
            r.isForward = isForward;

            queueOrSendMessage(H.LAUNCH_ACTIVITY, r);
        }
    
    }
}
```
ApplicationThread的scheduleRelaunchActivity()用来处理ActivityManagerService发出的SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION进程通信请求。
它主要把将要启动的目标Activity的信息封装成一个ActivityClientRecord对象，再以ActivityClientRecord对象为参数，调用queueOrSendMessage()函数
，向新创建应用进程的主线程消息队列发送一个类型为LAUNCH_ACTIVITY的消息。

### 31 ActivityThread.queueOrSendMessage(int what, Object obj)

```java
public final class ActivityThread{

    // if the thread hasn't started yet, we don't have the handler, so just
    // save the messages until we're ready.
    private final void queueOrSendMessage(int what, Object obj) {
        queueOrSendMessage(what, obj, 0, 0);
    }

    private final void queueOrSendMessage(int what, Object obj, int arg1) {
        queueOrSendMessage(what, obj, arg1, 0);
    }

    private final void queueOrSendMessage(int what, Object obj, int arg1, int arg2) {
        synchronized (this) {
            if (DEBUG_MESSAGES) Slog.v(
                TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
                + ": " + arg1 + " / " + obj);
            Message msg = Message.obtain();
            msg.what = what;
            msg.obj = obj;
            msg.arg1 = arg1;
            msg.arg2 = arg2;
            mH.sendMessage(msg);
        }
    }
    
}
```

将上一步传递过来的ActivityClientRecord对象封装成一个Message对象，并将消息LAUNCH_ACTIVITY发送。

### 32 H.handleMessage(Message msg)

```java
public final class H extends Handler{

  public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + msg.what);
            switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    //获取ActivityClientRecord对象
                    ActivityClientRecord r = (ActivityClientRecord)msg.obj;
                    //获取一个LoadedApk对象，并将它保存在ActivityClientRecord对象的成员变量packagInfo中。
                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo);
                    handleLaunchActivity(r, null);
                } break;
                case RELAUNCH_ACTIVITY: {
                    ActivityClientRecord r = (ActivityClientRecord)msg.obj;
                    handleRelaunchActivity(r, msg.arg1);
                } break;
                case PAUSE_ACTIVITY:
                    handlePauseActivity((IBinder)msg.obj, false, msg.arg1 != 0, msg.arg2);
                    maybeSnapshot();
                    break;
                case PAUSE_ACTIVITY_FINISHING:
                    handlePauseActivity((IBinder)msg.obj, true, msg.arg1 != 0, msg.arg2);
                    break;
                case STOP_ACTIVITY_SHOW:
                    handleStopActivity((IBinder)msg.obj, true, msg.arg2);
                    break;
                case STOP_ACTIVITY_HIDE:
                    handleStopActivity((IBinder)msg.obj, false, msg.arg2);
                    break;
                case SHOW_WINDOW:
                    handleWindowVisibility((IBinder)msg.obj, true);
                    break;
                case HIDE_WINDOW:
                    handleWindowVisibility((IBinder)msg.obj, false);
                    break;
                case RESUME_ACTIVITY:
                    handleResumeActivity((IBinder)msg.obj, true,
                            msg.arg1 != 0);
                    break;
                case SEND_RESULT:
                    handleSendResult((ResultData)msg.obj);
                    break;
                case DESTROY_ACTIVITY:
                    handleDestroyActivity((IBinder)msg.obj, msg.arg1 != 0,
                            msg.arg2, false);
                    break;
                case BIND_APPLICATION:
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    break;
                case EXIT_APPLICATION:
                    if (mInitialApplication != null) {
                        mInitialApplication.onTerminate();
                    }
                    Looper.myLooper().quit();
                    break;
                case NEW_INTENT:
                    handleNewIntent((NewIntentData)msg.obj);
                    break;
                case RECEIVER:
                    handleReceiver((ReceiverData)msg.obj);
                    maybeSnapshot();
                    break;
                case CREATE_SERVICE:
                    handleCreateService((CreateServiceData)msg.obj);
                    break;
                case BIND_SERVICE:
                    handleBindService((BindServiceData)msg.obj);
                    break;
                case UNBIND_SERVICE:
                    handleUnbindService((BindServiceData)msg.obj);
                    break;
                case SERVICE_ARGS:
                    handleServiceArgs((ServiceArgsData)msg.obj);
                    break;
                case STOP_SERVICE:
                    handleStopService((IBinder)msg.obj);
                    maybeSnapshot();
                    break;
                case REQUEST_THUMBNAIL:
                    handleRequestThumbnail((IBinder)msg.obj);
                    break;
                case CONFIGURATION_CHANGED:
                    handleConfigurationChanged((Configuration)msg.obj);
                    break;
                case CLEAN_UP_CONTEXT:
                    ContextCleanupInfo cci = (ContextCleanupInfo)msg.obj;
                    cci.context.performFinalCleanup(cci.who, cci.what);
                    break;
                case GC_WHEN_IDLE:
                    scheduleGcIdler();
                    break;
                case DUMP_SERVICE:
                    handleDumpService((DumpServiceInfo)msg.obj);
                    break;
                case LOW_MEMORY:
                    handleLowMemory();
                    break;
                case ACTIVITY_CONFIGURATION_CHANGED:
                    handleActivityConfigurationChanged((IBinder)msg.obj);
                    break;
                case PROFILER_CONTROL:
                    handleProfilerControl(msg.arg1 != 0, (ProfilerControlData)msg.obj);
                    break;
                case CREATE_BACKUP_AGENT:
                    handleCreateBackupAgent((CreateBackupAgentData)msg.obj);
                    break;
                case DESTROY_BACKUP_AGENT:
                    handleDestroyBackupAgent((CreateBackupAgentData)msg.obj);
                    break;
                case SUICIDE:
                    Process.killProcess(Process.myPid());
                    break;
                case REMOVE_PROVIDER:
                    completeRemoveProvider((IContentProvider)msg.obj);
                    break;
                case ENABLE_JIT:
                    ensureJitEnabled();
                    break;
                case DISPATCH_PACKAGE_BROADCAST:
                    handleDispatchPackageBroadcast(msg.arg1, (String[])msg.obj);
                    break;
                case SCHEDULE_CRASH:
                    throw new RemoteServiceException((String)msg.obj);
            }
            if (DEBUG_MESSAGES) Slog.v(TAG, "<<< done: " + msg.what);
        }
        
}
```
上述方法做了两件事情：

1 获取ActivityClientRecord对象，获取一个LoadedApk对象，并将它保存在ActivityClientRecord对象的成员变量packagInfo中。

>LoadedApk对象：每个Android应用都打包在一个APK文件中，APK文件包含应用的所有资源，LoadedApk对象就是用来描述一个已经被加载的APK文件。

2 调用handleLaunchActivity(r, null)方法来处理LAUNCH_ACTIVITY消息。

### 33 ActivityThread.handleLaunchActivity(ActivityClientRecord r, Intent customIntent) 

```java
public final class ActivityThread {
    
   private final void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();

        if (localLOGV) Slog.v(
            TAG, "Handling launch of " + r);
         //启动目标Activity组件
        Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            Bundle oldState = r.state;
            //将目标Activity组件状态设置为Resumed状态，表示他是系统当前激活的Activity
            handleResumeActivity(r.token, false, r.isForward);

            if (!r.activity.mFinished && r.startsNotResumed) {
                // The activity manager actually wants this one to start out
                // paused, because it needs to be visible but isn't in the
                // foreground.  We accomplish this by going through the
                // normal startup (because activities expect to go through
                // onResume() the first time they run, before their window
                // is displayed), and then pausing it.  However, in this case
                // we do -not- need to do the full pause cycle (of freezing
                // and such) because the activity manager assumes it can just
                // retain the current state it has.
                try {
                    r.activity.mCalled = false;
                    mInstrumentation.callActivityOnPause(r.activity);
                    // We need to keep around the original state, in case
                    // we need to be created again.
                    r.state = oldState;
                    if (!r.activity.mCalled) {
                        throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPause()");
                    }

                } catch (SuperNotCalledException e) {
                    throw e;

                } catch (Exception e) {
                    if (!mInstrumentation.onException(r.activity, e)) {
                        throw new RuntimeException(
                                "Unable to pause activity "
                                + r.intent.getComponent().toShortString()
                                + ": " + e.toString(), e);
                    }
                }
                r.paused = true;
            }
        } else {
            // If there was an error, for any reason, tell the activity
            // manager to stop us.
            try {
                ActivityManagerNative.getDefault()
                    .finishActivity(r.token, Activity.RESULT_CANCELED, null);
            } catch (RemoteException ex) {
            }
        }
    }
}
```

该函数主要做了两件事情：

```
1 调用performLaunchActivity(r, customIntent)启动目标Activity组件。
2 调用andleResumeActivity(r.token, false, r.isForward)将目标Activity组件状态设置为Resumed状态，表示他是系统当前激活的Activity
```

## 34 ActivityThread.performLaunchActivity(ActivityClientRecord r, Intent customIntent)

```java
public final class ActivityThread {

    private final Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // System.out.println("##### [" + System.currentTimeMillis() + "] ActivityThread.performLaunchActivity(" + r + ")");

        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        //获取Activity组件的包名以及类名
        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }

        //将目标Activity类文件加载到内存中，并且创建一个实例。
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            r.intent.setExtrasClassLoader(cl);
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
            if (localLOGV) Slog.v(
                    TAG, r + ": app=" + app
                    + ", appName=" + app.getPackageName()
                    + ", pkg=" + r.packageInfo.getPackageName()
                    + ", comp=" + r.intent.getComponent().toShortString()
                    + ", dir=" + r.packageInfo.getAppDir());

            if (activity != null) {
                //创建一个ContextImpl对象，它将作为Activity运行的上下文环境，通过它就可以访问特定的应用程序资源
                //，以及启动应用程序组件。
                ContextImpl appContext = new ContextImpl();
                appContext.init(r.packageInfo, r.token, this);
                appContext.setOuterContext(activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mConfiguration);
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.text + " with config " + config);
                //使用ContextImpl对象和ActivityClientRecord对象来初始化Activity组件
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstance,
                        r.lastNonConfigurationChildInstances, config);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstance = null;
                r.lastNonConfigurationChildInstances = null;
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                //Activity初始化完成后，调用mInstrumentation.callActivityOnCreate(activity, r.state)将Activity启动起来
                mInstrumentation.callActivityOnCreate(activity, r.state);
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
                r.stopped = true;
                if (!r.activity.mFinished) {
                    activity.performStart();
                    r.stopped = false;
                }
                if (!r.activity.mFinished) {
                    if (r.state != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }
                if (!r.activity.mFinished) {
                    activity.mCalled = false;
                    mInstrumentation.callActivityOnPostCreate(activity, r.state);
                    if (!activity.mCalled) {
                        throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPostCreate()");
                    }
                }
            }
            r.paused = true;

            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    }
    
}

```

这个函数正式开始Activity组件的初始化以及启动工作，我们来具体看看。

1 获取Activity组件的包名以及类名
2 将目标Activity类文件加载到内存中，并且创建一个实例。
3 创建一个ContextImpl对象，它将作为Activity运行的上下文环境，通过它就可以访问特定的应用程序资源，以及启动应用程序组件。使用ContextImpl对象和ActivityClientRecord对象
来初始化Activity组件。

4 Activity初始化完成后，调用mInstrumentation.callActivityOnCreate(activity, r.state)将Activity启动起来，我们来看看该函数的具体实现：

```java
public class Instrumentation {

    /**
     * Perform calling of an activity's {Activity#onCreate}
     * method.  The default implementation simply calls through to that method.
     * 
     * @param activity The activity being created.
     * @param icicle The previously frozen state (or null) to pass through to
     *               onCreate().
     */
    public void callActivityOnCreate(Activity activity, Bundle icicle) {
        if (mWaitingActivities != null) {
            synchronized (mSync) {
                final int N = mWaitingActivities.size();
                for (int i=0; i<N; i++) {
                    final ActivityWaiter aw = mWaitingActivities.get(i);
                    final Intent intent = aw.intent;
                    if (intent.filterEquals(activity.getIntent())) {
                        aw.activity = activity;
                        mMessageQueue.addIdleHandler(new ActivityGoing(aw));
                    }
                }
            }
        }
        
        activity.onCreate(icicle);
        
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    am.match(activity, activity, activity.getIntent());
                }
            }
        }
    }    

}
```

可以看到该函数最终调用了Activity的onCreate()方法。

### 35 Activity.onCreate(Bundle savedInstanceState) 

```java
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks {
        
    /**
     * Called when the activity is starting.  This is where most initialization
     * should go: calling { #setContentView(int)} to inflate the
     * activity's UI, using { #findViewById} to programmatically interact
     * with widgets in the UI, calling
     * { #managedQuery(android.net.Uri , String[], String, String[], String)} to retrieve
     * cursors for data being displayed, etc.
     * 
     * <p>You can call { #finish} from within this function, in
     * which case onDestroy() will be immediately called without any of the rest
     * of the activity lifecycle ({ #onStart}, { #onResume},
     * { #onPause}, etc) executing.
     * 
     * <p><em>Derived classes must call through to the super class's
     * implementation of this method.  If they do not, an exception will be
     * thrown.</em></p>
     * 
     * @param savedInstanceState If the activity is being re-initialized after
     *     previously being shut down then this Bundle contains the data it most
     *     recently supplied in {onSaveInstanceState}.  <b><i>Note: Otherwise it is null.</i></b>
     */
    protected void onCreate(Bundle savedInstanceState) {
        mVisibleFromClient = !mWindow.getWindowStyle().getBoolean(
                com.android.internal.R.styleable.Window_windowNoDisplay, false);
        mCalled = true;
    }
    
        /**
         * Finds a view that was identified by the id attribute from the XML that
         * was processed in {@link #onCreate}.
         *
         * @return The view if found or null otherwise.
         */
        public View findViewById(int id) {
            return getWindow().findViewById(id);
        }
    
        /**
         * Set the activity content from a layout resource.  The resource will be
         * inflated, adding all top-level views to the activity.
         * 
         * @param layoutResID Resource ID to be inflated.
         */
        public void setContentView(int layoutResID) {
            getWindow().setContentView(layoutResID);
        }
    
        /**
         * Set the activity content to an explicit view.  This view is placed
         * directly into the activity's view hierarchy.  It can itself be a complex
         * view hierarhcy.
         * 
         * @param view The desired content to display.
         */
        public void setContentView(View view) {
            getWindow().setContentView(view);
        }
    
        /**
         * Set the activity content to an explicit view.  This view is placed
         * directly into the activity's view hierarchy.  It can itself be a complex
         * view hierarhcy.
         * 
         * @param view The desired content to display.
         * @param params Layout parameters for the view.
         */
        public void setContentView(View view, ViewGroup.LayoutParams params) {
            getWindow().setContentView(view, params);
        }
    
        /**
         * Add an additional content view to the activity.  Added after any existing
         * ones in the activity -- existing views are NOT removed.
         * 
         * @param view The desired content to display.
         * @param params Layout parameters for the view.
         */
        public void addContentView(View view, ViewGroup.LayoutParams params) {
            getWindow().addContentView(view, params);
        }
}
```

全面人民万众瞩目，翘首以盼的onCreate()函数终于来到了我们面前。<img src="https://github.com/guoxiaoxing/emoji/raw/master/emoji/d_ku.png" width="30" height="30" align="bottom"/>

我们平时常见的Activity代码：

```java
public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        findViewById(R.id.btn_application).setOnClickListener(this);
        findViewById(R.id.btn_system).setOnClickListener(this);
        findViewById(R.id.btn_program).setOnClickListener(this);
    }

    @Override
    public void onClick(View view) {
        switch (view.getId()) {
            case R.id.btn_application: {
                Intent intent = new Intent(MainActivity.this, ApplicationActivity.class);
                startActivity(intent);
            }
            break;
            case R.id.btn_system: {
                Intent intent = new Intent(MainActivity.this, SystemActivity.class);
                startActivity(intent);
            }
            break;
            case R.id.btn_program: {
                Intent intent = new Intent(MainActivity.this, ProgramActivity.class);
                startActivity(intent);
            }
            break;
        }
    }
}

```
onCreate()主要用来加载用户界面，以及对用户界面上的组件进行初始化，

