---
title: Battery Historian 电量分析工具介绍
date: 2018-05-22 17:41:27
tags: 
- Android
- 电池
- 功耗  



---




### Battery Historian 简介  ###
> Battery Historian 是一个这样的的工具：可以在 Android 5.0 Lollipop（API 级别21）及更高版本的 Android 设备上检测与电池相关的信息和事件，而在此期间，该设备没有插上电源。它允许应用程序开发人员在时间轴上可视化系统和应用级别的事件，并使用平移和缩放功能，在设备最后一次完全充电之后，可以轻松地查看各种聚合统计信息，可以选择一个应用程序，检查所选择的应用程序对电池指标的影响。此外，它还允许对两个错误报告进行 A/B 比较，突出显示了关键电池相关指标的差异。

**Battery Historian源码地址：https://github.com/google/battery-historian**

<!--more-->

### 配置方法  ###
Battery Historian 的运行需要很多环境的支持，要做很多配置，官网介绍了两个方法，一种是通过 Docker_百度百科 使用官网提供的已配置好的容器，另外一种就是老老实实自己配各种环境，第一种的Using Docker 看似较为简单，但有很多坑 ...

#### Using Docker ####
Using Docker（通过 Docker 来间接使用 Battery Historian）

Docker下载地址：https://docs.docker.com/engine/installation/

**你会惊奇地发现，Docker 只支持 Mac 和 Windows 10 ，哈哈，让你不用 Mac 开发：**  

#### 通过各种配置后从源码构建 ####
**（1）安装 Go 语言**  
  下载地址：https://golang.org/doc/install

安装配置环境变量  

![](https://i.imgur.com/ABhSXWU.png)

    检查是否安装成功：cmd 执行 “go version”
![](https://i.imgur.com/iJtJrz8.png)

**（2）安装 Python**  
下载：https://www.python.org/   
【注意仅支持 python 2.7，python3.0改变很大】  
安装配置环境变量  

    检查是否安装成功：cmd 执行 “python –V”【注意是大写V】  
![](https://i.imgur.com/VRGTsL6.png)


**（3）安装Git**  

下载地址：https://git-scm.com/  

    安装检查是否安装成功：cmd 执行 “git version”  
![](https://i.imgur.com/9DX49TA.png)


**（4）下载 Battery Historian 源码**  
cmd 执行下载命令  

    “go get -d -u github.com/google/battery-historian/...”
【注意最后有三个点】

运行setup，需要下载想关联的软件包或者js

    cmd 执行“go run setup.go”
【第一次执行要下载，时间会久一些，只需要执行一次，以后就快些】  
![](https://i.imgur.com/U4KtFbx.png)

    cmd 执行“go run cmd/battery-historian/battery-historian.go” 运行battery-historian  

![](https://i.imgur.com/7saAR0K.png)

**登录查看**  

    登录网址 http://localhost:9999 查看是否运行  
![](https://i.imgur.com/SNvsQXl.png)

### 导出手机的 Bugreport 文件   ###

    cmd 执行“adb bugreport > bugreport.txt”  


上传 bugreport.txt 文件至 http://localhost:9999  
![](https://i.imgur.com/KMzJby5.png)

### 通过Battery Historian 分析   ###

![](https://i.imgur.com/l8WpSaF.png)
  


**FAQ：**

1. Question：battery historian 上传Bugreport文件之后 Submit按钮不显示  
 ![](https://i.imgur.com/IQuGNLx.png)  
   Answer：这是由于国内无法访问外网导致，有一些在线JS无法加载出来，可以通过购买vpn代理或者安装插件解决。
