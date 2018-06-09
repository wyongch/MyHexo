---
title: Android Doze 省电原理
date: 2018-05-22 18:48:24
tags:
- Android
- 省电模式
- Doze
categories: Android

---
## Doze 简介 ##

在M（6.0）版本之后，在设备或者app空闲的状态下，android引入了一个新的省电模式 Doze  
Doze: 提高设备空闲时的休眠效率
如果用户设备屏幕关闭，没有充电，并且静置一段时间，设备将会进入到doze模式，尝试保持系统休眠状态.

<!-- more -->

In N, Doze are enhanced  
 stationary(M/N)  
 non-stationary(N)  

Introduction about Doze&AppStandby  


参考文档
https://developer.android.com/training/monitoring-device-state/doze-standby.html  
https://developer.android.com/about/versions/nougat/android-7.0-changes.html


## Doze原理 ##
在doze模式下，系统会尝试通过限制应用程序访问网络和CPU密集型服务来节省电量。 它也阻止应用程序访问网络并推迟作业，同步和标准定时器。

系统会定期退出doze，让应用程序完成延期活动。 在此维护期间，系统将运行所有未完成的同步，作业和定时任务，并允许应用程序访问网络。

![](https://i.imgur.com/xois1Kp.png)



“DeviceIdle” 添加到系统服务，实现 Doze 特性.   
DeviceIdleController.java  Doze的具体实现：通过状态机切换不同的状态，并且发送广播通知相关服务做一些限制操作。

![](https://i.imgur.com/SHkUuOi.png)

### DeviceIdleController服务启动 ###
SystemServer 服务中启动DeviceIdleController服务  
\frameworks\base\services\java\com\android\server\SystemServer.java

    private void startOtherServices() {
        ……
        mSystemServiceManager.startService(DeviceIdleController.class);
        ……
    }

\frameworks\base\services\core\java\com\android\server\DeviceIdleController.java






### Doze模式状态切换流程 ###

**如下图所示：**
![](https://i.imgur.com/JpzjStW.png)



## Doze 两种模式 ##

在N版本上有两种Doze模式：deep  mode and light mode. 在M版本上只有 deep mode 一种.  
Deep idle mode & light idle mode
深度模式需要静止一段时间，但light模式不需要。
深度模式和light模式可以同时在设备上运行。.
设备首先进入light空闲模式，然后进入深度空闲模式
当设备进入深度空闲模式（空闲状态）时，light模式被忽略。 


### deep moden ###
在deep moden下，app有如下限制:  
1. 网络访问被暂停。  
2. 系统忽略唤醒锁。  
3. 标准唤醒定时器(包括setExact() and setWindow()) 被推迟到下一个维护window.  
4. 如果你需要在Doze模式下设置定时器，请使用 setAndAllowWhileIdle() or setExactAndAllowWhileIdle().  
5. 使用 setAlarmClock() 在系统退出Doze状态能够正常使用   
6. 系统不执行Wi-Fi扫描。  
7. 系统不允许同步任务运行。  
8. 系统不允许JobScheduler运行。  




### light mode ###

The following restrictions apply to your apps while in light idle mode:
Network access is suspended.
The system does not allow sync adapters to run.
The system does not allow JobScheduler to run.




