---
title: Android  电池消耗百分比计算方法
date: 2018-05-22 18:47:23
tags:
---




# 概述 #
> 在android系统中，用户可以通过Settings中的电池选项，可以看到各个应用所消耗的电池百分比。这部分google原生支持，ODM厂商会在该基础之上做一些定制修改

<!--more-->

电池消耗一般可以分为两大类，一种是软件消耗，一种是硬件器件消耗。
主要涉及的代码如下：
frameworks\base\core\res\res\xml\power_profile.xml 配置各个器件不同状态的电流值
packages\apps\Settings\src\com\android\settings\fuelgauge\PowerUsageSummary.java 具体的电流计算方法

# 计算公式 #



## 应用程序 ##



应用程序的电池消耗分为CPU消耗、wake lock消耗、数据传输消耗、GPS消耗、wifi连接消耗，其中wifi比较特殊，除了分散在各个应用中外，在电池统计项中也有一项专门的wifi项，这个也比较好理解，像数据传输、GPS都是应用使用的，而wifi除了应用使用外，即使没有任何应用使用，也还需要保持连接，这部分的消耗就归类到单独的wifi项里了。

### CPU消耗 ###

![](https://i.imgur.com/knWUeW5.png)

其中CPU频率在power_profile.xml中的”cpu.speeds”中填写，可填写多个，各个频率下对应的cpu功率则在”cpu.active”中填写，顺序上必须与”cpu.speeds”一致。CPU时间指的是在指定的频率下运行的时间。

### Wake lock消耗 ###

![](https://i.imgur.com/vJUvHvI.png)

其中wake lock功率在power_profile.xml中的”cpu.awake”中填写。

### 数据传输消耗 ###

![](https://i.imgur.com/kLdFCjV.png)

其中WiFi功率在power_profile.xml中的”wifi.active”填写，移动数据功率在power_profile.xml中的” radio.active”填写。

### GPS消耗 ###

![](https://i.imgur.com/YKm1NcM.png)

其中GPS功率在power_profile.xml中的”gps.on”中填写。

### Wi-Fi连接消耗 ###

![](https://i.imgur.com/nIOvIEv.png)

其中WiFi连接功率在power_profile.xml中的”wifi.on”填写。

所以

![](https://i.imgur.com/FvQLWiz.png)

对于每个应用，Settings采用UID来遍历每个系统中的所有应用，我们可以通过adb shell ps来查看USER栏，这个就是UID的name。如下：

	USER     PID   PPID  VSIZE  RSS     WCHAN    PC         NAME
	root      158   2     0      0     c00cba00 00000000 S pvr_workqueue
	root      184   2     0      0     c00cba00 00000000 S omaplfb
	root      208   2     0      0     c015f410 00000000 S flush-179:0
	system    215   141   573012 88196 ffffffff 40040730 S system_server
	log       291   1     732    284   c02dfad8 400c84f8 S /system/bin/logwrapper
	bluetooth 292   291   18256  3744  ffffffff 400f8650 S /system/bin/btld
	system    332   141   498532 72836 ffffffff 40041444 S com.android.systemui
	app_59    417   141   468868 47104 ffffffff 40041444 S com.huawei.inputmethod.hwpal
	log       457   1     732    284   c02dfad8 400d94f8 S /system/bin/logwrapper
	bluetooth 458   457   2144   1196  c014b78c ffff0520 S /system/bin/bluetoothd
	radio     459   141   498380 54744 ffffffff 40041444 S com.android.phone
	bluetooth 466   141   457940 33844 ffffffff 40041444 S com.broadcom.bt.app.system
	app_61    488   141   530612 86720 ffffffff 40041444 S com.huawei.android.launcher

对于一般的应用，它们的UID Name都是app_开头，后面接数字。这个表现在电池消耗列表中就是一个单独的应用。而对于USER是root的，就归类为Android OS，对于USER是system的，就归类为Android System（Android 系统），对于USER是mediaserver的，就归类为Mediaserver（媒体服务器）。而USER是wi-fi和bluetooth的应用，则是统计到器件消耗，在下一章会介绍。


## 器件消耗 ##
### 语言通话 ###

![](https://i.imgur.com/L9HesmV.png)

其中通话功率在power_profile.xml中的”radio.active”填写。

### 屏幕 ###

![](https://i.imgur.com/AVKdG7V.png)

计算屏幕功耗时把屏幕亮度分为5个等级，分别为0、1、2、3、4。其中亮屏功耗在power_profile.xml中的”screen.on”填写，背光最亮时的功耗在”screen.full”填写。

### Wi-Fi ###

![](https://i.imgur.com/w0XTl5y.png)

其中是计算应用程序时，UID是Process.WIFI_UID的应用程序所消耗的功耗。应用使用的WiFi连接时间是计算应用程序时，所有应用所占据的WiFi连接时间的综合。WiFi连接功率在power_profile.xml中的”wifi.on”填写。

### 蓝牙 ###

![](https://i.imgur.com/dZ5zUBU.png)

其中是计算应用程序时，UID是Process.BLUETOOTH_GID的应用程序所消耗的功耗。蓝牙功耗在power_profile.xml中的”bluetooth.on”填写，蓝牙ping功耗在”bluetooth.at”中填写。

### 插卡待机耗电量 ###

![](https://i.imgur.com/DXl0WkU.png)

其中待机功耗在power_profile.xml中的”radio.on”中填写，可以填写多列对应不同信号强度。搜索信号功耗在”radio.scanning”中填写。



### 灭屏待机耗电量 ###

![](https://i.imgur.com/XAw7q95.png)

其中CPU待机功耗在power_profile.xml中的”cpu.idle”中填写。注意：如果是平板电脑，此项显示为“平板电脑待机”。

## 统计方法 ##
手机总的耗电量就是上面各个子项耗电量的总和。

![](https://i.imgur.com/nGW3pRG.png)

在Settings的电池选项中，最多会列出10项最耗电的子项，对于耗电量在5mAH以下或耗电百分比在1%以下的，就不会显示在列表中。所以总和是有可能小于100%的。另外也并不是所有的器件耗电都有统计，如camera就没有统计camera器件的耗电量。所以耗电量只是一个估计值，仅供参考。

#索引#
Power_profile.xml文件说明
（如果文件中未配置某项，计算耗电量时会认为这一项的值为0。）

	cpu.idle
	cpu.awake
	cpu.active
	wifi.scan
	wifi.on
	wifi.active
	gps.on
	bluetooth.on
	bluetooth.active
	bluetooth.at
	screen.on
	radio.on
	radio.scanning
	radio.active
	screen.full
	dsp.audio
	dsp.video
	cpu.speeds
	battery.capacity





# Dump batteryinfo #
进入adb shell，通过dumpsys batteryinfo可以查看更详细的电池信息，之前提到过，setting里面的电池信息中CPU唤醒时间只包含了java层应用程序的唤醒时间，对于kernel层的唤醒时间无法看到，通过dumpsys batteryinfo命令，就可以看到kernel层哪些进程在保持cpu的唤醒状态。原理是通过文件节点/proc/wakelocks来统计的。

