---
title: Android死机重启问题分析思路
date: 2018-05-15 16:16:46
tags: 
- Android  
- 死机重启
- 功耗
- 性能
categories: Android


---



## SWT简介 ##

> SWT是Software Watchdog的缩写，在android系统中为了监控systemservers是否处于正常运行，加入了SWT线程来监控SystemServer重要线程/Service的运行情况。判断如果阻塞超过60秒，就会把系统重启，来保证系统恢复正常状态 

概述：android系统中死机跟重启是有联系的，系统级的服务发生异常导致系统无响应，android系统中通过watchdog机制，将系统服务的无响应转换成重启，保证系统正常运行。

<!--more-->

## 死机现象：   ##
   点击屏幕，power键无响应

## 死机原因： ##
逻辑行为异常  
1. 逻辑判断错误  
2. 逻辑设计错误

逻辑卡顿(block)  
1. 死循环 (Deadloop)  
2. 死锁 (Deadlock)



## watchdog机制原理： ##
watchdong机制中主要有两种方式去check systemserver   
1 通过Service注册的monitor去check

2 通过handler发送msg到重要的loop线程来check是否阻塞

![](https://i.imgur.com/Vt610dl.png)

watchdog监控的系统线程  


## watchdog 执行流程 ##

初始化：添加好所以要监控的线程和service

开 始：watchdog 本身是一个实现runnable接口的线程，开始运行后，每分钟check一次初始化中添加的线程以及service是否有回应

半分钟：半分钟后check系统是否有卡顿，如果卡住则dump system_server的backtrace，否则重新计时

一分钟：一分钟后检查系统是否有卡住，如果卡住则第二次dump trace并且kill 掉systemserver，否则重新计时

如下图

![](https://i.imgur.com/4oJNbqT.png)


## 死机重启问题分析思路 ##

了解了wachdog机制就可以按照如下步骤分析死机重启问题

![](https://i.imgur.com/tZRcuvP.png)

通过以上思路一般的死机重启问题都能分析定位出来，有些比较特殊的情况就要结合代码逻辑来分析。
