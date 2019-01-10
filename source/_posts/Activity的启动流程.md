---
title: Activity的启动流程
date: 2017-03-24
categories: "Android"
tags: "源码"
---
# 概述
昨天已经差不多写好这篇博文了，只剩下最后的总结没写，想着今早再补上去，结果忘记保存了（写的那么长竟然也能忘记保存，我挺佩服自己的）。。。按理说没保存也能在缓存里面找回来的，可是今早开电脑的时候手贱点开了360清理了一下，现在心里可苦可苦了。还好只是篇文章，不是什么重要的文件，血的教训，标记一下。今天重新写过。
关于Activity的启动流程我也是学习了挺久的了，看了很多文章。网上也有大神觉得这个很简单，不过对于我来讲还是挺难理解的，因为里面涉及到了IPC（进程间的通讯）和binder的原理，对于这块其实我还不很熟，所以先学习了IPC和binder再来研究Activity的启动流程，学习嘛，一定要学到关键的地方，要了解原理，不然很快就忘了。现在开始吧
<!-- more -->
# 具体分析
通常我们从一个Activity跳转到另外一个Activity，我们知道首先回调周期方法的是onPause()，之后才开始LaunchActicity。先从startActivity(intent)开始，看何时才会调用onPause()，何时调用scheduleLaunchActivity()。
先看一个在网上找的流程图，还真的有些复杂。
![](http://oxr4g4c3v.bkt.clouddn.com/Activity%E7%9A%84%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.jpg)

### 源码分析
**Activity.class**：
```java
//其实`startActivity(intent)`就是`startActivityForResult(intent,requestCode)`，只是把requestCode置为-1而已。其实啊，code小于0都可以的。
@Override
   public void startActivity(Intent intent, @Nullable Bundle options) {
       if (options != null) {
           startActivityForResult(intent, -1, options);
       } else {
           // Note we want to go through this call for compatibility with
           // applications that may have overridden the method.
           startActivityForResult(intent, -1);
       }
   }
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
           @Nullable Bundle options) {
       // ... ...
	
           options = transferSpringboardActivityOptions(options);
		//Instrumentation是Activity的成员变量，它用来监控应用程序和系统的交互；
           Instrumentation.ActivityResult ar =
               mInstrumentation.execStartActivity(
                   this, mMainThread.getApplicationThread(), mToken, this,
                   intent, requestCode, options);
           if (ar != null) {
			//MainThread也是Activity的成员变量，都是通过attach()方法在ActivityThread创建Activity的时候传递给Activity作为成员变量。
               mMainThread.sendActivityResult(
                   mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                   ar.getResultData());
           }
		
       // ... ...
	
   }
```
看下Instrumentation.class的execStartActivity方法：
```java
public ActivityResult execStartActivity(
           Context who, IBinder contextThread, IBinder token, Activity target,
           Intent intent, int requestCode, Bundle options) {
		
	//whoThread：ApplicationThread，一个binder对象，作为参数将会传递到AcitivityMangerService远程服务中，后面会用此作为远程回调，在AtivityThread中对Activity的生命周期//的控制都是在这个类里面。
       IApplicationThread whoThread = (IApplicationThread) contextThread;
	
	// ... ...
	
	//ActivityManagerNative.getDefault()是获取ActivityManagerService
	//而ActivityManagerNative.getDefault().startActivity就是一个进程间的通信
       int result = ActivityManagerNative.getDefault().startActivity(whoThread, who.getBasePackageName(), intent,
                       intent.resolveTypeIfNeeded(who.getContentResolver()),
                       token, target != null ? target.mEmbeddedID : null,
                       requestCode, 0, null, options);
					
       checkStartActivityResult(result, intent);
           
       
       // ... ...
   }
// 检测结果
   public static void checkStartActivityResult(int res, Object intent) {
       if (res >= ActivityManager.START_SUCCESS) {
           return;
       }
       switch (res) {
		//开启Acticity时出现的各种错误
           case ActivityManager.START_INTENT_NOT_RESOLVED:
           case ActivityManager.START_CLASS_NOT_FOUND:
               // 假如没有在AndroiMananifest注册的话，就会在这里抛出异常
               if (intent instanceof Intent && ((Intent)intent).getComponent() != null)
                   throw new ActivityNotFoundException(
                           "Unable to find explicit activity class "
                           + ((Intent)intent).getComponent().toShortString()
                           + "; have you declared this activity in your AndroidManifest.xml?");
               throw new ActivityNotFoundException(
                       "No Activity found to handle " + intent);
           case ActivityManager.START_PERMISSION_DENIED:
               throw new SecurityException("Not allowed to start activity "
                       + intent);
           case ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT:
               throw new AndroidRuntimeException(
                       "FORWARD_RESULT_FLAG used while also requesting a result");
           case ActivityManager.START_NOT_ACTIVITY:
               throw new IllegalArgumentException(
                       "PendingIntent is not an activity");
           case ActivityManager.START_NOT_VOICE_COMPATIBLE:
               throw new SecurityException(
                       "Starting under voice control not allowed for: " + intent);
           case ActivityManager.START_VOICE_NOT_ACTIVE_SESSION:
               throw new IllegalStateException(
                       "Session calling startVoiceActivity does not match active session");
           case ActivityManager.START_VOICE_HIDDEN_SESSION:
               throw new IllegalStateException(
                       "Cannot start voice activity on a hidden session");
           case ActivityManager.START_CANCELED:
               throw new AndroidRuntimeException("Activity could not be started for "
                       + intent);
           default:
               throw new AndroidRuntimeException("Unknown error code "
                       + res + " when starting " + intent);
       }
   }
```
进入ActivityManagerService.class的startActivity方法：
```java
@Override
   public final int startActivity(IApplicationThread caller, String callingPackage,
           Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
           int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
       return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
               resultWho, requestCode, startFlags, profilerInfo, bOptions,
               UserHandle.getCallingUserId());
   }
@Override
   public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
           Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
           int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
       enforceNotIsolatedCaller("startActivity");
       userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
               userId, false, ALLOW_FULL_ONLY, "startActivity", null);
       // TODO: Switch to user app stacks here.
       return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
               resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
               profilerInfo, null, null, bOptions, false, userId, null, null);
   }
```
ActivityStarter.class的startActivityMayWait方法：
```java
final int startActivityMayWait(IApplicationThread caller, int callingUid,
           String callingPackage, Intent intent, String resolvedType,
           IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
           IBinder resultTo, String resultWho, int requestCode, int startFlags,
           ProfilerInfo profilerInfo, IActivityManager.WaitResult outResult, Configuration config,
           Bundle bOptions, boolean ignoreTargetSecurity, int userId,
           IActivityContainer iContainer, TaskRecord inTask) {
		
		int res = startActivityLocked(caller, intent, ephemeralIntent, resolvedType,
                   aInfo, rInfo, voiceSession, voiceInteractor,
                   resultTo, resultWho, requestCode, callingPid,
                   callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                   options, ignoreTargetSecurity, componentSpecified, outRecord, container,
                   inTask);
}
final int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
           String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
           IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
           IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
           String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
           ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
           ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container,
           TaskRecord inTask) {
           // 验证intent、Class、Permission等
           // 保存将要启动的Activity的Record
           err = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
                   true, options, inTask);
           return err;
     }
  
  private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
           IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
           int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask) {
           // 检查将要启动的Activity的launchMode和启动Flag
           // 根据launcheMode和Flag配置task
           final boolean dontStart = top != null && mStartActivity.resultTo == null
               && top.realActivity.equals(mStartActivity.realActivity)
               && top.userId == mStartActivity.userId
               && top.app != null && top.app.thread != null
               && ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
               || mLaunchSingleTop || mLaunchSingleTask);
           // 举一个例子 SingleTop
           if (dontStart) {
               top.deliverNewIntentLocked(
                   mCallingUid, mStartActivity.intent, mStartActivity.launchedFromPackage);
              // Don't use mStartActivity.task to show the toast. We're not starting a new activity
              // but reusing 'top'. Fields in mStartActivity may not be fully initialized.
              mSupervisor.handleNonResizableTaskIfNeeded(
                   top.task, preferredLaunchStackId, topStack.mStackId);
              return START_DELIVERED_TO_TOP;
           }
           mTargetStack.startActivityLocked(mStartActivity, newTask, mKeepCurTransition, mOptions);
           if (mDoResume) {
			//进入ActivityStack的resumeTopActivityInnerLocked()方法
                 mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                       mOptions);
           }
     }
```
进入ActvityStack.class的startActivityLocked方法：
```java
// 任务栈历史栈配置
     final void startActivityLocked(ActivityRecord r, boolean newTask, boolean keepCurTransition,
           ActivityOptions options) {
           if (!r.mLaunchTaskBehind && (taskForIdLocked(taskId) == null || newTask)) {
               // 加入栈顶 管理栈
               insertTaskAtTop(rTask, r);
               // 管显示
               mWindowManager.moveTaskToTop(taskId);
           }
           if (!newTask) {
               // 不是一个新的Task
               task.addActivityToTop(r);
               r.putInHistory();
               addConfigOverride(r, task);
           }
     }
  
  private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
          // Find the first activity that is not finishing.
          final ActivityRecord next = topRunningActivityLocked();
          // The activity may be waiting for stop, but that is no longer
          // appropriate for it.
          mStackSupervisor.mStoppingActivities.remove(next);
          mStackSupervisor.mGoingToSleepActivities.remove(next);
          next.sleeping = false;
          mStackSupervisor.mWaitingVisibleActivities.remove(next);
         if (mResumedActivity != null) {
             if (DEBUG_STATES) Slog.d(TAG_STATES,
                   "resumeTopActivityLocked: Pausing " + mResumedActivity);
             pausing |= startPausingLocked(userLeaving, false, next, dontWaitForPause);
         }
     }
     final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
           ActivityRecord resuming, boolean dontWait) {
           if (prev.app != null && prev.app.thread != null) {
           if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Enqueueing pending pause: " + prev);
           try {
               EventLog.writeEvent(EventLogTags.AM_PAUSE_ACTIVITY,
                       prev.userId, System.identityHashCode(prev),
                       prev.shortComponentName);
               mService.updateUsageStats(prev, false);
               // 暂停Activity
			//prev.app.thread是指主线程
               prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
                       userLeaving, prev.configChangeFlags, dontWait);
           }
           completePauseLocked(false, resuming);
     }
```
看下ActivityThead的内部类ApplicationThread.class的schedulePauseActivity方法：
```java
public final void schedulePauseActivity(IBinder token, boolean finished,
               boolean userLeaving, int configChanges, boolean dontReport) {
               sendMessage(
                   finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
                   token,
                   (userLeaving ? USER_LEAVING : 0) | (dontReport ? DONT_REPORT : 0),
                   configChanges,
                   seq);
     }
     public void handleMessage(Message msg) {
           switch (msg.what) {
               ...
           case PAUSE_ACTIVITY: {//调用onPause()
                   Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
                   SomeArgs args = (SomeArgs) msg.obj;
                   handlePauseActivity((IBinder) args.arg1, false,
                           (args.argi1 & USER_LEAVING) != 0, args.argi2,
                           (args.argi1 & DONT_REPORT) != 0, args.argi3);
                   maybeSnapshot();
                   Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
          } break;
      }
      private void handlePauseActivity(IBinder token, boolean finished,
           boolean userLeaving, int configChanges, boolean dontReport, int seq) {
           //... 
           performPauseActivity(token, finished, r.isPreHoneycomb(), "handlePauseActivity");
           //...
           // Tell the activity manager we have paused.
           if (!dontReport) {
               try {
                   ActivityManagerNative.getDefault().activityPaused(token);
               } catch (RemoteException ex) {
                   throw ex.rethrowFromSystemServer();
               }
           }
      }
      final Bundle performPauseActivity(IBinder token, boolean finished,
           boolean saveState, String reason) {
           ActivityClientRecord r = mActivities.get(token);
           return r != null ? performPauseActivity(r, finished, saveState, reason) : null;
      }
      final Bundle performPauseActivity(ActivityClientRecord r, boolean finished,
           boolean saveState, String reason) {
           // ...
           performPauseActivityIfNeeded(r, reason);
      }
      private void performPauseActivityIfNeeded(ActivityClientRecord r, String reason) {
           // ...
		//这里进入Instrumentation，再调用Activity的performPause()方法，这里很简单就不贴代码了
           mInstrumentation.callActivityOnPause(r.activity);
      }
```
终于，终于，让我们在Activity.class看到 onPause()。
```java
final void performPause() {
        mDoReportFullyDrawn = false;
        mFragments.dispatchPause();
        mCalled = false;
        onPause();
        mResumed = false;
        if (!mCalled && getApplicationInfo().targetSdkVersion
                >= android.os.Build.VERSION_CODES.GINGERBREAD) {
            throw new SuperNotCalledException(
                    "Activity " + mComponent.toShortString() +
                    " did not call through to super.onPause()");
        }
        mResumed = false;
    }
```
ActivityManagerNative.getDefault().activityPaused(token) ——> ActivityManagerService的activityPaused(token)方法
```java
@Override
   public final void activityPaused(IBinder token) {
       final long origId = Binder.clearCallingIdentity();
       synchronized(this) {
           ActivityStack stack = ActivityRecord.getStackLocked(token);
           if (stack != null) {
               stack.activityPausedLocked(token, false);
           }
       }
       Binder.restoreCallingIdentity(origId);
   }
```
ActivityStackactivityPausedLocked方法：
```java
final void activityPausedLocked(IBinder token, boolean timeout){
        completePauseLocked(true, null);
    }
    private void completePauseLocked(boolean resumeNext, ActivityRecord resuming) {
        ActivityRecord prev = mPausingActivity;
        if (resumeNext) {
            final ActivityStack topStack = mStackSupervisor.getFocusedStack();
            mStackSupervisor.resumeFocusedStackTopActivityLocked(topStack, prev, null);
        }
    }
```
ActivityStackSupervisor中的resumeFocusedStackTopActivityLocked()方法
```java
boolean resumeFocusedStackTopActivityLocked(
        ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
        targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
    }
```
ActivityStack中的resumeTopActivityUncheckedLocked()方法
```java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        result = resumeTopActivityInnerLocked(prev, options);
    }
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        mStackSupervisor.startSpecificActivityLocked(next, true, true);
    }
```
ActivityStackSupervisor中的startSpecificActivityLocked()方法
```java
void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);
        r.task.stack.setLaunchTime(r);
        if (app != null && app.thread != null) {
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
                    // Don't add this if it is a platform component that is marked
                    // to run in multiple processes, because this is actually
                    // part of the framework so doesn't make sense to track as a
                    // separate apk in the process.
                    app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                            mService.mProcessStats);
                }
                // 启动Activity
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }
            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }
    final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {
            // scheduleLaunchActivity 启动
            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(task.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);
    }
```
**ActivityTheadc.class**:
```java
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
            updateProcessState(procState, false);
            ActivityClientRecord r = new ActivityClientRecord();
            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.referrer = referrer;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            r.persistentState = persistentState;
            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;
            r.startsNotResumed = notResumed;
            r.isForward = isForward;
            r.profilerInfo = profilerInfo;
            r.overrideConfig = overrideConfig;
            updatePendingConfiguration(curConfig); 
			
            //到这里，真的启动了
            sendMessage(H.LAUNCH_ACTIVITY, r);
   }
   
   private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
   
	   // If we are getting ready to gc after going to the background, well 
	   // we are back active so skip it. 
	   //代码省略 
	   //根据类名实例化Activity 
	   Activity a = performLaunchActivity(r, customIntent); 
	   //代码省略
   }
   
   private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // ......
		Activity activity = null;
		
        //根据类名实例化Activity  
        activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
        // ... ...
        
        activity.attach(appContext, this, getInstrumentation(), r.token,
                r.ident, app, r.intent, r.activityInfo, title, r.parent,
                r.embeddedID, r.lastNonConfigurationInstances, config,
                r.referrer, r.voiceInteractor, window);
        // ... ...
    }
```
到了attach，这里就贴个图
![](http://oxr4g4c3v.bkt.clouddn.com/Activity%E7%9A%84%E5%90%AF%E5%8A%A8.jpg)

# 成员分析
- ActivityManagerService 组件通信系统核心管理类 （ActivityManagerNative）IPC通信，管理Activity的生命周期
- Instrumentation 是android系统中启动Activity的一个实际操作类，也就是说Activity在应用进程端的启动实际上就是Instrumentation执行的
- ActivityStackSupervisor 管理整个手机的Activity任务栈
- ActivityStack Activity栈（任务栈）存放Activity，后进先出
- ActivityThread Activity的入口是onCreate方法，Android上一个应用的入口是ActivityThread。和普通的Java类一样有一个main方法。用于控制与管理一个应用进程的主线程的操作，包括管理与处理activity manager发送过来的关于activities、广播以及其他的操作请求。

# 总结
Activity的启动流程一般是通过调用startActivity或者是startActivityForResult来开始的，会在Instrumentation类中实现具体启动的细节（涉及到进程间的通信），与ActivityManagerService进行进程间交互。ActivityManagerService接收到应用进程创建Activity的请求之后会执行初始化操作，解析启动模式，保存请求信息等一系列操作。ActivityManagerService通过Binder进程间通信机制将当前系统栈顶的Activity执行onPause操作，ActivityManagerService将执行创建Activity的通知告知ActivityThread，然后通过反射机制创建出Activity对象，并执行Activity的onCreate方法，onStart方法，onResume方法。


   
