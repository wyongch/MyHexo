---
title: 理解 Android Context
date: 2018-06-28 19:05:32
tags:
- Android
- Context

---


> 作为一名开发人员Context 在程序设计中经常被使用到，如果你还没听说，哈哈哈……说明你是一个假程序员，我们都知道目前操作系统与用户
进行交互形式是以事件驱动的，Context就是对用户与操作系统交互过程这一场景的抽象，有了这个概念 Context就好理解了。

<!--more-->

# 什么是Context #
Context中文翻译是：上下文，环境，场景；我认为在程序中比较准确的翻译是场景。
一个Context就是一个场景，一个场景就是用户与操作系统的一个交互过程。比如：打开相机，场景就包括前台相机预览界面和后台数据操作等，
刷微博，场景就包括前台信息流显示，以及后台网络连接，数据处理等。

从程序的角度看，操作系统把用户与操作系统的交互过程抽象成Context，这样我们就很好理解为什么Activity和Service都是继承自Context
，我们可以简单把Activity是一个前台有界面场景的交互；而Service是一个无界面的后台场景，从代码的继承关系也可以说明这一点：

Context相关类继承关系如图所示：

![](https://i.imgur.com/mcqRcGY.png)

类文件说明：

1. Context ：操作系统对用户一个操作场景的抽象类
2. ContextImpl：Context类的具体实现类，应用程序中调用的Content方法，具体实现均来源于该类
3. ContextWrapper： 顾名思义 Context的包装类，为了方便开发者使用
4. ContextThemeWrapper： 提供了与主题相关的接口


# Application对应的Context #

每一个应用程序在启动时都会创建一个Application对象，默认Application对象是应用程序包名，用户可以重载默认的Application对象。
方法是在AndroidManifest.xml的Application标签中声明一个新的Application名称。

Application创建ContextImp流程如下：

![](https://i.imgur.com/ZEPSQVx.png)

首先AMS通过binder远程调用到 ActivityThread 中的bindApplication（）方法，该方法参数包括两种，一种是ApplicationInfo，这是
实现Parcelable接口的数据类，意味着这个对象是AMS创建的，通过IPC传递到ActivityThread中，另一种是其他相关参数。

        public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableBinderTracking, boolean trackAllocation,
                boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                CompatibilityInfo compatInfo, Map<String, IBinder> services, Bundle coreSettings) {

            if (services != null) {
                // Setup the service cache in the ServiceManager
                ServiceManager.initServiceCache(services);
            }

            setCoreSettings(coreSettings);

            AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            data.debugMode = debugMode;
            data.enableBinderTracking = enableBinderTracking;
            data.trackAllocation = trackAllocation;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;
            sendMessage(H.BIND_APPLICATION, data);
        }

在bindApplication方法中，会用以上两种参数构造一个本地AppBindData数据类，然后去调用handlerBindApplication（）方法。
在调用handlerBindApplication的时候，AppBindData对象中的info还是空值，然后会使用data.info = getPackageInfoNoCheck()
方法为info赋值，

        data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);

而这个方法的内部实际上会根据AppBindData中的ApplicationInfo中的mPackageName创建一个PackageInfo对象，
并把这个对象保存为ActivityThread类的全局对象。显然一个应用程序中所有Activity或者Application对象对应的mPackage都是一样的，
即为包的名称，所有ActivityThread中会产生一个全局的PackageInfo对象。



            Application app = data.info.makeApplication(data.restrictedBackupMode, null);
            mInitialApplication = app;


            try {
                ClassLoader cl = this.getClassLoader();
                if(!this.mPackageName.equals("android")) {
                    this.initializeJavaContextClassLoader();
                }

                ContextImpl appContext = ContextImpl.createAppContext(this.mActivityThread, this);
                app = this.mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext);
                appContext.setOuterContext(app);
            } catch (Exception var10) {
                if(!this.mActivityThread.mInstrumentation.onException(app, var10)) {
                    throw new RuntimeException("Unable to instantiate application " + appClass + ": " + var10.toString(), var10);
                }
            }

# Activity 对应的Context #

启动Activity时，AMS会通过IPC调用到ActivityThread的scheduleLaunchActivity（）方法，该方法包含两种参数。
种是ActivityInof，这是一个实现了Parcelable接口的数据类，表示该类是AMS创建的，并通过IPC传递到ActivityThread；另一种是其他一些参数。

        @Override
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

            sendMessage(H.LAUNCH_ACTIVITY, r);
        }

scheduleLaunchActivity（）会根据以上两种参数构造一个本地ActivityRecord数据类，ActivityThread内部会为每一个Activity
创建一个对应的ActivityRecord对象，并使用这些数据对象来管理Activity。

接着会调用handleLauncherActivity（），然后再调用performLaunchActivity（），该方法中创建ContextImp的代码如下：

     try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
          ……
         ）

r.packageInfo是从哪儿来的？答案如下

    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }
        ……
     }

即在performLaunchActivity（）开始执行时，首先为 r.packageInfo变量赋值，getPackageInfo（）方法的执行逻辑和getPackageInfoNoCheck（）
基本相同。r.PackageInfo 对象的PackageInfo对象和Application对应的packageInfo对象是同一个。

流程图如下：

![](https://i.imgur.com/m40sOIp.png)

# Service 对应的Context #

启动Service时，AMS首先会通过IPC调用到ActivityThread的scheduleCreateService（）方法，该方法也包含两种参数。第一种是ServiceInfo，
这是实现Parcelable接口的数据类，意味着这个对象是AMS创建的，通过IPC传递到ActivityThread中，另一种是其他相关参数。

在scheduleCreateService方法中，会使用以上两种参数构造一个CreateServiceData数据对象，ActivityThread会为其包含的每一个
Service创建对应的ServiceInfo数据对象，并且通过该对象管理Service。

接着会执行handleCreateService（）方法，其中创建ContextImpl对象代码如下：

            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);

与前两节相同，CreateAppContext第二个参数packageInfo赋值代码如下：

        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);

赋值代码同样使用了getPackageInfoNoCheck（），这就意味着Service对应的Context对象内部的mPackageInfo与Activity，Application中完全相同。

# 总结 #

以上三节可以看出，创建Context对象过程基本上是相同的，包括代码结构也十分类似，所不同的仅仅是针对Application，Activity，Service使用了不同的数据对象，可总结为如下表：


<table>
  <tr>
    <th width=25%, bgcolor= > 类名</th>
    <th width=25%, bgcolor=> 远程数据类</th>
    <th width=25%, bgcolor=> 本地数据类 </th>
    <th width=25%, bgcolor=> 赋值方式 </th>
  </tr>
  <tr>
    <td> Application </td>
    <td> ApplicationInfo </td>
    <td> AppBindData </td>
    <td> getPackageInfoNoCheck() </td>
  </tr>
  <tr>
    <td> Activity </td>
    <td> ActivityInfo </td>
    <td> ActivityRecord </td>
    <td> getPackageInfo() </td>
  </tr>
  <tr>
    <td> Service </td>
    <td> ServiceInfo </td>
    <td> CreateServiceData </td>
    <td> getPackageInfoNoCheck() </td>
  </tr>
</table>


从上面三节我们也可以看出，实际上一个应用包含Context的个数为：

Context个数=Service个数+Activity个数+1（Application）

