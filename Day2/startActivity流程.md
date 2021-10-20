## startActivity启动流程（针对于 Android 11.0）

### 1、进入Activity  startActivity：

```java
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (mIntent != null && mIntent.hasExtra(AutofillManager.EXTRA_RESTORE_SESSION_TOKEN)
            && mIntent.hasExtra(AutofillManager.EXTRA_RESTORE_CROSS_ACTIVITY)) {
       ...
    }
    if (options != null) {
      //进入此方法
        startActivityForResult(intent, -1, options);
    } else {
        // Note we want to go through this call for compatibility with
        // applications that may have overridden the method.
        startActivityForResult(intent, -1);
    }
}
```



### 2、进入Activity  startActivityForResult：

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
         if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            Instrumentation.ActivityResult ar =
              //进入此方法拿到一个ActivityResult对象
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
             
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
            // TODO Consider clearing/flushing other event sources and events for child windows.
            
         }   
        ...
```



### 3、进入Instrumentation execStartActivity,Instrumentation 类负责 **Application** 和 **Activity** 的建立和生命周期控制:

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    Uri referrer = target != null ? target.onProvideReferrer() : null;
  ...
    
    try {
        intent.migrateExtraStreamToClipData(who);
        intent.prepareToLeaveProcess(who);
        //获取ATMS启动-startActivity
        int result = ActivityTaskManager.getService().startActivity(whoThread,
                who.getOpPackageName(), who.getAttributionTag(), intent,
                intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```



### 4  、进入ActivityTaskManager  getService():

```java

public static IActivityTaskManager getService() {
    return IActivityTaskManagerSingleton.get();
}
  //通过模板单例创建IActivityTaskManager，而IActivityTaskManager是通过AIDL方式获取的
 
 @UnsupportedAppUsage(trackingBug = 129726065)
    private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
            new Singleton<IActivityTaskManager>() {
                @Override
                protected IActivityTaskManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                    return IActivityTaskManager.Stub.asInterface(b);
                }
            };
```

IActivityTaskManager是一个AIDL接口，通过动态代理方式获取，这里通过Binder IPC机制获取和ActivityTaskManagerService构建连接，因为ActivityTaskManagerService实现了IActivityTaskManager接口。

```java
public class ActivityTaskManagerService extends IActivityTaskManager.Stub {...}
```



### 5、其中IActivityTaskManager接口定义了需要实现的startActivity方法：

```java
public final int startActivity(IApplicationThread caller, String callingPackage,
        String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
        String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
        Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
            resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}
```

进入startActivityAsUser

``` java
@Override
    public int startActivityAsUser(IApplicationThread caller, String callingPackage,
            String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
            Bundle bOptions, int userId) {
        return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
                true /*validateIncomingUser*/);
    }

```

 在此处切换到用户app堆栈

```java
 private int startActivityAsUser(IApplicationThread caller, String callingPackage,
            @Nullable String callingFeatureId, Intent intent, String resolvedType,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
        assertPackageMatchesCallingUid(callingPackage);
        enforceNotIsolatedCaller("startActivityAsUser");   

userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
            Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

    // 在此处切换到用户app堆栈
    return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
            .setCaller(caller)
            .setCallingPackage(callingPackage)
            .setCallingFeatureId(callingFeatureId)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setStartFlags(startFlags)
            .setProfilerInfo(profilerInfo)
            .setActivityOptions(bOptions)
            .setUserId(userId)
            .execute();

}
```



### 6、进入ActivityStarter obtainStarter，通过ActivityStartController控制并且代理Activity启动，obtainStarter用于配置和执行启动活动的启动器：

```java
  ActivityStarter obtainStarter(Intent intent, String reason) {  
    return mFactory.obtain().setIntent(intent).setReason(reason);
}
```



### 7、进入ActivityStarter，它是专门负责一个 Activity 的启动操做。它的主要做用包括解析 Intent、建立 ActivityRecord、若是有可能还要建立 TaskRecordflex，执行Activity启动请求：



```java
     int execute() {
        try {
            // Refuse possible leaked file descriptors
            if (mRequest.intent != null && mRequest.intent.hasFileDescriptors()) {
                throw new IllegalArgumentException("File descriptors passed in Intent");
            }  

          final LaunchingState launchingState;
          synchronized (mService.mGlobalLock) {
            final ActivityRecord caller = ActivityRecord.forTokenLocked(mRequest.resultTo);
            launchingState = mSupervisor.getActivityMetricsLogger().notifyActivityLaunching(
                    mRequest.intent, caller);
        }

        //如果调用者尚未解决该活动，我们愿意在此解决。 
       // 如果调用者已经在这里持有 WM 锁，并且我们需要检查动态 Uri 权限，
        //那么我们被迫假设这些权限被拒绝以避免死锁。
        if (mRequest.activityInfo == null) {
            mRequest.resolveActivity(mSupervisor);
        }

        int res;
        synchronized (mService.mGlobalLock) {
            final boolean globalConfigWillChange = mRequest.globalConfig != null
                    && mService.getGlobalConfiguration().diff(mRequest.globalConfig) != 0;
            final ActivityStack stack = mRootWindowContainer.getTopDisplayFocusedStack();
            if (stack != null) {
                stack.mConfigWillChange = globalConfigWillChange;
            }
            if (DEBUG_CONFIGURATION) {
                Slog.v(TAG_CONFIGURATION, "Starting activity when config will change = "
                        + globalConfigWillChange);
            }

            final long origId = Binder.clearCallingIdentity();

            res = resolveToHeavyWeightSwitcherIfNeeded();
            if (res != START_SUCCESS) {
                return res;
            }
            //请求处理
            res = executeRequest(mRequest);

            Binder.restoreCallingIdentity(origId);
          。。。
    } finally {
        onExecutionComplete();
    }
}
```

```java
/**
 * Activity启动之前需要做那些准备？
 * 通常的Activity启动流程将通过 startActivityUnchecked 到 startActivityInner。
 */
private int executeRequest(Request request) {
    ...
    //Activity的记录
    ActivityRecord sourceRecord = null;
    ActivityRecord resultRecord = null;
    ...
    //Activity stack(栈)管理
    mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
            request.voiceInteractor, startFlags, true /* doResume */, checkedOptions, inTask,
            restrictedBgActivity, intentGrants);

    ...
    return mLastStartActivityResult;
}
```

### 8、进入startActivityUnchecked:

```java
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, Task inTask,
            boolean restrictedBgActivity, NeededUriGrants intentGrants) {
    ....
    try {
        ...
        result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, doResume, options, inTask, restrictedBgActivity, intentGrants);
    } finally {
        ...
    }

    postStartActivityProcessing(r, result, startedActivityRootTask);

    return result;
}
```

### 9、进入startActivityInner()：

```java
int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, Task inTask,
        boolean restrictedBgActivity, NeededUriGrants intentGrants) {
    setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
            voiceInteractor, restrictedBgActivity);

    //计算启动模式
    computeLaunchingTaskFlags();
    computeSourceRootTask();
    //设置启动模式
    mIntent.setFlags(mLaunchFlags);

    ...

    // 关键点来了
    mRootWindowContainer.resumeFocusedStacksTopActivities(
                    mTargetRootTask, mStartActivity, mOptions, mTransientLaunch);
    ...

    return START_SUCCESS;
}
```

在ActivityStarter中通过ActivityRecord的启动模式启动Activity,ActivityRecord是一个记录Activity启动方式，appToken,ActivityInfo,包名等信息的类，其中在ActivityStarter类中传入了一个ActivityRecord参数，在ActivityStarter startActivityInner根据设置的不同launchMode在ActivityStack创建或复用Activity

### 10、进入关键类RootWindowContainer， 它是窗口容器的根容器，它内部实现了屏幕亮度调整，屏幕休眠时间调整等逻辑：

```java
boolean resumeFocusedStacksTopActivities(
        ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

    if (!mStackSupervisor.readyToResume()) {
        return false;
    }

    boolean result = false;
    if (targetStack != null && (targetStack.isTopStackInDisplayArea()
            || getTopDisplayFocusedStack() == targetStack)) {
      //确保Activity在栈顶
        result = targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
    }

    for (int displayNdx = getChildCount() - 1; displayNdx >= 0; --displayNdx) {
        boolean resumedOnDisplay = false;
        final DisplayContent display = getChildAt(displayNdx);
        for (int tdaNdx = display.getTaskDisplayAreaCount() - 1; tdaNdx >= 0; --tdaNdx) {
            final TaskDisplayArea taskDisplayArea = display.getTaskDisplayAreaAt(tdaNdx);
            for (int sNdx = taskDisplayArea.getStackCount() - 1; sNdx >= 0; --sNdx) {
                final ActivityStack stack = taskDisplayArea.getStackAt(sNdx);
                final ActivityRecord topRunningActivity = stack.topRunningActivity();
                if (!stack.isFocusableAndVisible() || topRunningActivity == null) {
                    continue;
                }
                if (stack == targetStack) {
                   // 简单地更新 targetStack 的结果，因为 targetStack 有
                     // 已经在上面恢复了。 不想再次恢复它，尤其是在
                     // 在某些情况下，如果应用程序进程被启动会导致第二次启动失败
                     // 销毁。
                    resumedOnDisplay |= result;
                    continue;
                }
                if (taskDisplayArea.isTopStack(stack) && topRunningActivity.isState(RESUMED)) {
                   // 从 MoveTaskToFront 开始任何挥之不去的应用程序转换
                    // 操作，但只考虑该显示上的顶部任务和堆栈。
                    stack.executeAppTransition(targetOptions);
                } else {
                    resumedOnDisplay |= topRunningActivity.makeActiveIfNeeded(target);
                }
            }
        }
        if (!resumedOnDisplay) {
            // 在没有有效活动的情况下（例如设备刚刚启动或启动器
             // 崩溃）可能没有在显示器上恢复任何内容。
             // 明确聚焦堆栈中的顶部活动将确保至少到home
             // 活动开始和恢复，没有发生递归。
            final ActivityStack focusedStack = display.getFocusedStack();
            if (focusedStack != null) {
                result |= focusedStack.resumeTopActivityUncheckedLocked(target, targetOptions);
            } else if (targetStack == null) {
                result |= resumeHomeActivity(null /* prev */, "no-focusable-task",
                        display.getDefaultTaskDisplayArea());
            }
        }
    }

    return result;
}
```

```java
@GuardedBy("mService")
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    if (mInResumeTopActivity) {
        // Don't even start recursing.
        return false;
    }

    boolean result = false;
    try {
        // Protect against recursion.
        mInResumeTopActivity = true;
        result = resumeTopActivityInnerLocked(prev, options);

      // 恢复top Activity时，可能需要暂停top Activity（比如返回锁屏。我们在b中抑制了正常的暂停逻辑，因为top Activity是在最后恢复的。我们调用{@ 在此处再次链接 ActivityStackSupervisor#checkReadyForSleepLocked} 以确保发生任何必要的暂停逻辑。如果无论锁定屏幕如何都会显示 Activity，则跳过对 {@link ActivityStackSupervisor#checkReadyForSleepLocked} 的调用。
        final ActivityRecord next = topRunningActivity(true /* focusableOnly */);
        if (next == null || !next.canTurnScreenOn()) {
            checkReadyForSleep();
        }
    } finally {
        mInResumeTopActivity = false;
    }

    return result;
}
```

### 11、在ActivityStackSupervisor中的realStartActivityLocked方法创建ClientTransaction：

```java
boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
        boolean andResume, boolean checkConfig) throws RemoteException {

    ...
            //创建 Activity 启动的 transaction
            final ClientTransaction clientTransaction = ClientTransaction.obtain(
                    proc.getThread(), r.appToken);

            final boolean isTransitionForward = r.isTransitionForward();
            clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                    System.identityHashCode(r), r.info,
                    mergedConfiguration.getGlobalConfiguration(),
                    mergedConfiguration.getOverrideConfiguration(), r.compat,
                    r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                    r.getSavedState(), r.getPersistentSavedState(), results, newIntents,
                    r.takeOptions(), isTransitionForward,
                    proc.createProfilerInfoIfNeeded(), r.assistToken, activityClientController,
                    r.createFixedRotationAdjustmentsIfNeeded(), r.shareableActivityToken,
                    r.getLaunchedFromBubble()));

            // Set desired final state.
            final ActivityLifecycleItem lifecycleItem;
            if (andResume) {
                lifecycleItem = ResumeActivityItem.obtain(isTransitionForward);
            } else {
                lifecycleItem = PauseActivityItem.obtain();
            }
            clientTransaction.setLifecycleStateRequest(lifecycleItem);

            // Schedule transaction.
            mService.getLifecycleManager().scheduleTransaction(clientTransaction);
     
    ...
    return true;
}
```



创建IApplicationThread，IApplicationThread是一个Binder远程调用接口，在ActivityThread的静态内部类ApplicationThread就实现了此接口：


```java
/** Target client. */
private IApplicationThread mClient;

public void schedule() throws RemoteException {
    mClient.scheduleTransaction(this);
}
```


```java
//这里咱们传入的是一个启动 Activity 的 transaction
void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    final IApplicationThread client = transaction.getClient();
    //注释1
    transaction.schedule();
    if (!(client instanceof Binder)) {
        // 如果 client 不是Binder的实例，则它是一个远程调用，此时可以安全地回收该对象。
        // 在ActivityThread中的客户端上执行transaction后，将回收用于本地调用的所有对象。
        transaction.recycle();
    }
}
```



### 12、最终在ActivityThread 中ApplicationThread实现了Binder远程调用接口：

```java
public final class ActivityThread extends ClientTransactionHandler
        implements ActivityThreadInternal {
    ...
    private class ApplicationThread extends IApplicationThread.Stub {
        ...
        @Override
        public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
            ActivityThread.this.scheduleTransaction(transaction);
        }
        ...
    }
    ...
}

```

然后通过Hander发送消息，消息Flag是**ActivityThread.H.EXECUTE_TRANSACTION**，然后通过消息处理任务，进入TransactionExecutor.execute()，通过execute执行任务：

```java
public void execute(ClientTransaction transaction) {
    ...
    executeCallbacks(transaction);
    ...
}
```

### 13、经过一系列处理最终进入TransactionExecutor，进入到performLifecycleSequence方法中执行Activity生命周期的方法：

```java
/** Transition the client through previously initialized state sequence. */
private void performLifecycleSequence(ActivityClientRecord r, IntArray path,
        ClientTransaction transaction) {
    final int size = path.size();
    for (int i = 0, state; i < size; i++) {
        state = path.get(i);
        
        switch (state) {
            case ON_CREATE:
                mTransactionHandler.handleLaunchActivity(r, mPendingActions,
                        null /* customIntent */);
                break;
            case ON_START:
                mTransactionHandler.handleStartActivity(r, mPendingActions,
                        null /* activityOptions */);
                break;
            case ON_RESUME:
                mTransactionHandler.handleResumeActivity(r, false /* finalStateRequest */,
                        r.isForward, "LIFECYCLER_RESUME_ACTIVITY");
                break;
            case ON_PAUSE:
                mTransactionHandler.handlePauseActivity(r, false /* finished */,
                        false /* userLeaving */, 0 /* configChanges */, mPendingActions,
                        "LIFECYCLER_PAUSE_ACTIVITY");
                break;
            case ON_STOP:
                mTransactionHandler.handleStopActivity(r, 0 /* configChanges */,
                        mPendingActions, false /* finalStateRequest */,
                        "LIFECYCLER_STOP_ACTIVITY");
                break;
            case ON_DESTROY:
                mTransactionHandler.handleDestroyActivity(r, false /* finishing */,
                        0 /* configChanges */, false /* getNonConfigInstance */,
                        "performLifecycleSequence. cycling to:" + path.get(size - 1));
                break;
            case ON_RESTART:
                mTransactionHandler.performRestartActivity(r, false /* start */);
                break;
            default:
                throw new IllegalArgumentException("Unexpected lifecycle state: " + state);
        }
    }
}
```

具体的生命周期方法的实现在ActivityThread中实现，因为它实现了ClientTransactionHandler接口。

## 总结流程

当我们调用startActivity时，流程会进入到Instrumentation的execSatartActivity方法，在这个方法中做了一件事：

-   通过Java单例模板模式创建IActivityTaskManager(老版本的Android版本是ActivityManager)远程接口，而这个接口是通过动态代理的方式获取的。

然后进入ActivityTaskManagerService(老版本的Android版本是ActivityManagerService),通过ActivityTaskManagerService中的startActivity和startActivityAsUser最终先进入一个关键类ActivityStarter，在这个类中做了以下几件事：
+ 1、计算和设置启动模式；

+ 2、根据启动模式在ActivityStack创建或复用Activity;

+ 3、创建RootWindowContainer，在RootWindowContainer进行下一步操作；

  然后在RootWindowContainer创建了一个ActivityStack(这里有一个小问题先提下，在Android11.0中，只存在ActivityTask和ActivityRecord之间的关系，而低版本Android则存在ActivityTask、TaskRecord和ActivityRecord的关系)，在ActivityStack创建了ActivityTaskSupervisor，而ActivityTaskSupervisor这个类是和ClientTranscation构建联系的类，而ClientTranscation的和抽象类ClientTransactionHandler构建联系的一个关键类，此外，ClientTranscation中定义了远程调用接口IApplicationThread相关方法，ClientTransactionHandler是ActivityThread的实现类，里面定义了Activity各个生命周期启动的方法，最后会在ApplicationThread中通过远程调用IApplicationThread接口拿到启动的Activity，然后通过Handler发送消息开启走生命周期的任务。
