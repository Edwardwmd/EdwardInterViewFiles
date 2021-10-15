## setContentView实现流程

+   1 、在源码Activity的setContentView：

    ```java
    public void setContentView(@LayoutRes int layoutResID) {
        //重点来了
        getWindow().setContentView(layoutResID);
        //创建并设置ActionBar,
        initWindowDecorActionBar();
    }
    ```

+   2、Activity.getWindow  获取phonewindow：

```java
public Window getWindow() {
    return mWindow;
}
```

+   3、Activity.attach()  实际上整个 Android 系统中 Window 只有一个实现类，就是 PhoneWindow：

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken,
        IBinder shareableActivityToken) {
    mFragments.attachHost(null /*parent*/);
        //重点来了对PhoneWindow进行实例化
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(mWindowControllerCallback);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        ...
        //调用setWindowManager
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        ...
}
```

+   4、Window.setWindowManager   创建了 WindowManagerImpl 对象，windowmanager和Window关联起来：

    ```java
    public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
            boolean hardwareAccelerated) {
        mAppToken = appToken;
        mAppName = appName;
        mHardwareAccelerated = hardwareAccelerated;
        if (wm == null) {
            //通过Binder机制来获取WMS
            wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        }
        //再次创建WindowManager
        mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
    }
    
    public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
        return new WindowManagerImpl(mContext, parentWindow);
    }
    ```

+   5、PhoneWindow.setContentView    Activity 将 setContentView 的操作交给了 PhoneWindow：

    

    ```java
    @Override
        public void setContentView(int layoutResID) {
            //注意：当主题属性等结晶化时，可在安装窗口装饰的过程中设置功能内容转换。
            //在这种情况发生之前，不要检查功能。
            if (mContentParent == null) {
                //调用 installDecor 初始化 DecorView 和 mContentParent。
                installDecor();
            } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
                mContentParent.removeAllViews();
            }
    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            //mLayoutInflater.inflate(layoutResID, mContentParent); 从指定的xml资源展开新的视图层次结构。调用 setContentView 传入的布局添加到 mContentParent 中
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
    
    
    
    
    private void installDecor() {
            mForceDecorInstall = false;
            //初始化DecorView
            if (mDecor == null) {
                mDecor = generateDecor(-1);
                mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
                mDecor.setIsRootNamespace(true);
                if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                    mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
                }
            } else {
                mDecor.setWindow(this);
            }
            //初始化mContentParent
            if (mContentParent == null) {
                mContentParent = generateLayout(mDecor);
                ...
            }
        }
    ```

由源码可知，PhoneWindow 中默认有一个 DecorView（实际上是一个 FrameLayout），在 DecorView 中默认自带一个 mContentParent（ViewGroup）。我们自己实现的布局是被添加到 mContentParent 中的，因此经过 setContentView 之后。

Activity 执行到 onCreate 时并不可见，只有执行完 onResume 之后 Activity 中的内容才是屏幕可见状态。造成这种现象的原因就是，onCreate 阶段只是初始化了 Activity 需要显示的内容，而在 onResume 阶段(当界面要与用户进行交互时，会调用ActivityThread的handleResumeActivity方法)才会将 PhoneWindow 中的 DecorView 真正的绘制到屏幕上。

+   6、ActivityThread.handleResumeActivity    在 ActivityThread 的 handleResumeActivity 中，会调用 WindowManager 的 addView 方法将 DecorView 添加到 WMS(WindowManagerService) 上： 

    ```java
    @Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
            ...
            ViewManager wm = a.getWindowManager();
            ...
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    //WindowManger 的 addView 结果有两个:DecorView 被渲染绘制到屏幕上显示； DecorView 可以接收屏幕触摸事件
                    wm.addView(decor, l);
                } else {
                    a.onWindowAttributesChanged(l);
                }
            }
            ...
    }
    ```

ViewManager是一个接口，WindowManager是个接口同时又实现了ViewManager接口，而WindowManagerImpl又实现了WindowManager，这是只捋清楚这个方法的层级，清楚后放到一边继续查看WindowManagerImpl中的addview方法.

+   7、 WindowManagerGlobal.addView ：

    ```java
    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow, int userId) {
        ...
        ViewRootImpl root;
        View panelParentView = null;
    
        synchronized (mLock) {
            ...
            root = new ViewRootImpl(view.getContext(), display);
    
            //WindowMangerGlobal 是一个单例 在 addView 方法中，创建了一个最关键的 ViewRootImpl 对象。
            view.setLayoutParams(wparams);
    
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
    
            //最后执行此操作，因为它会发出消息开始执行操作
            try {
                //然通过 root.setView 方法将 view 添加到 WMS 中
                root.setView(view, wparams, panelParentView, userId);
            } catch (RuntimeException e) {
                ...
            }
        }
    }
    ```

8、ViewRootImpl.setView   ：

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
        int userId) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
            ...
            int res; /* = WindowManagerImpl.ADD_OKAY; */
            //requestLayout 是刷新布局的操作，调用此方法后 ViewRootImpl 所关联的 View 也执行 measure -> layout -> draw 操作，确保在 View 被添加到 Window 上显示到屏幕之前，已经完成测量和绘制操作。
            requestLayout();
            InputChannel inputChannel = null;
            if ((mWindowAttributes.inputFeatures
                    & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                inputChannel = new InputChannel();
            }
            mForceDecorViewVisibility = (mWindowAttributes.privateFlags
                    & PRIVATE_FLAG_FORCE_DECOR_VIEW_VISIBILITY) != 0;
            try {
                ...
                //调用 mWindowSession 的 addToDisplay 方法将 View 添加到 WMS 中
                res = mWindowSession.addToDisplayAsUser(mWindow, mSeq, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(), userId, mTmpFrame,
                        mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                        mAttachInfo.mDisplayCutout, inputChannel,
                        mTempInsets, mTempControls);
                setFrame(mTmpFrame);
            } catch (RemoteException e) {
            ...
            } finally {
                if (restore) {
                    attrs.restore();
                }
            }
            
    }
}
```

总结：setContView的过程全程是通过PhoneWindow实现，WMS是幕后操作：
1、在ActivityThread初始化时，handleLauncherActivity–>performLauncherActivity 进入attach()方法中初始化了PhoneWindow和创建本地WinowManager

2、构建PhoneWindow的联接（判断WinowManager是否为null,如果为null就通过Binder机制从WMS中获取），获取WindowManager的实现类WindowManagerImpl；

3、通过调用PhoneWindom的setContentView先创建一个默认DecorView,调用LayoutInflate.inflate()将布局填充到DecorView中。

4、布局是在onResume方法后才可见，因为绘制流程在onResume之后，所以在onResume中WindowManager 通过addView方法将DecorView添加到WMS，addView在WindowManager 的实现类WindowManagerImpl才做具体实现。

5、在WindowManagerImpl创建了ViewRootImpl,通过ViewRootImpl setView将view添加到WMS中。

6、在ViewRootImpl setView()中带哦用reqyuestLayout()实现布局测量、摆放、绘制，最后页面会显示。

