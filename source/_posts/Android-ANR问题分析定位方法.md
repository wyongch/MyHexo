---
title: Android ANR问题分析定位方法
date: 2018-05-23 14:07:51
tags:
- Android
- Activity
categories: Android
---




Crash类的基本上都是由各类未处理的JAVA异常导致，打开下载回来的日志压缩包，里面一般都有名为 system_app_crash@xxxxxxxxxxxx.txt 的日志，直接开启即可看到对应的调用堆栈，一般都能直接看到出错的代码具体位置以及直接原因（根据JAVA异常类的含义来推断），属于最好分析解决的一类。麻烦一点的可能就是异常是从native层通过JNI抛出的异常，此时只能继续通过JNI继续跟踪到抛出对应异常的位置，分析原因并解决。

> Crash 主要分为Java层的和Native层，java层的crash 一般比较好分析，通过 log 和调用trace 可以准确的定位到具体是哪个类哪一行出现的问题，一般有空指针或者除数为零的原因导致；Native crash 一般是jni抛出的异常，需要通过 命令和对应的错误地址 定位是哪里出现问题



<!--more-->



# Java 层 crash #

例如，打开crash_ log_ 16__ 2018_0327_141801后可以看到如下日志：



	  ----- timezone:GMT  
	01-01 08:29:40.320553  1346  1346 E AndroidRuntime: FATAL EXCEPTION: main  
	01-01 08:29:40.320553  1346  1346 E AndroidRuntime: <font color=red>Process: com.android.settings, PID: 1346 </font>  
	01-01 08:29:40.320553  1346  1346 E AndroidRuntime: <font color=red>java.lang.RuntimeException: Unable to start activity 
	ComponentInfo{com.android.settings/com.android.settings.Settings$DateTimeSettingsActivity}: java.lang.NullPointerException: Attempt to invoke virtual method 'void android.support.v7.preference.Preference.onPrepareForRemoval()' on a null object reference  </font>  
	01-01 08:29:40.320553  1346  1346 E AndroidRuntime: 	at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2818)  
	01-01 08:29:40.320553  1346  1346 E AndroidRuntime: 	at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2883)  
	01-01 08:29:40.320553  1346  1346 E AndroidRuntime: 	at android.app.ActivityThread.-wrap12(ActivityThread.java)  
	01-01 08:29:40.320553  1346  1346 E AndroidRuntime: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1611)  
	01-01 08:29:40.320553  1346  1346 E AndroidRuntime: 	at android.os.Handler.dispatchMessage(Handler.java:110)  
	01-01 08:29:40.320553  1346  1346 E AndroidRuntime: 	at android.os.Looper.loop(Looper.java:203)  
	01-01 08:29:40.320553  1346  1346 E AndroidRuntime: 	at android.app.ActivityThread.main(ActivityThread.java:6387)  
	01-01 08:29:40.320553  1346  1346 E AndroidRuntime: 	at java.lang.reflect.Method.invoke(Native Method)  
	01-01 08:29:40.320553  1346  1346 E AndroidRuntime: 	at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:1084)  
	01-01 08:29:40.320553  1346  1346 E AndroidRuntime: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:945)  


这几句log可以很清楚的看出是settings模块中发生了空指针异常，调用了一个无效的方法。
对应的类名，代码行数，全都打印出来了，找出原因直接到对应的位置修改即可。


> 对于Crash类问题，要尽量找到出现异常的根因，而不是简单的加个try catch将异常捕获，除非是异常不是由本模块抛出的或者其它设计上的原因。

# Native 层异常 Tombstone #

Tombstone主要见于native层的出错，可以认为是native层的crash，一般是应用自身的lib库报错，也有可能是framework层的报错（多见于各类service），相对于java层Crash，Tombstone由于lib都是编译后的代码，出错信息无法直接对应于代码，且不会像JAVA那样有分类相对比较明细的异常类型，分析起来较为困难。
　　tombstone的日志主要是看SYSTEM_TOMBSTONE@xxxxxxxx.txt.gz压缩包或者system_app_native_crash@xxxxxxxxxxxx.txt，其中后者可能没有（一般是应用自身的lib库出错才会有，framework的lib库基本没有），两者包含的信息基本是一致，后者可能多个app的信息。


可以看出两个日志差不多，native_crash有应用的基本信息（如包名版本号等），而SYSTEM_TOMBSTONE则多出硬件还有内核编译的版本信息，以及CPU 浮点寄存器和堆栈内的数据（大部分情况下无用处）。


首先关注tombstone日志如图的部分，其中第一行依次标明了出错模块的进程id、线程id、线程名称、进程名称；第二行最为关键，signal之后跟的数字代表了出错类型，根据其含义就能大致推断出引起错误的直接原因（signal机制，是在软件层次对中断机制的一种模拟，内核让某进程意识到某特殊事情发生了。强迫进程去执行相应的信号处理函数。至于信号的来源可能来自硬件如按下键盘或者硬件故障，可能来自其他进程，可能来自自己进程，这里我并不需要关注这些，感兴趣的可以自行网上搜索相关资料）。

![](https://i.imgur.com/pnQuvpB.png)

![](https://i.imgur.com/FGAGk5N.png)

![](https://i.imgur.com/mCEeylb.png)



## 常见报错类型——signal 11 ##

![](https://i.imgur.com/57b71hG.png)

Signal 11——SIGSEGV
最为常见的出错类型，通常是访问了错误的内存地址造成，如访问野指针、空指针、已释放的内存地址、数组越界访问等等，signal之后，都会跟上fault addr 0xXXXXXXXX字样，对应访问出错的地址，如果是0x0或者很小的数字，基本可以确定就是访问空指针。如上图所示的的微信，就是典型的访问了空指针。对于fault addr很小的情况，则多为使用空结构指针去访问成员变量所致。

![](https://i.imgur.com/4Vjbdr2.png)

## 常见报错类型——signal 6 ##

Signal 6——SIGABRT
第二常见的类型，是主动报错然后结束自身进程，通常用于程序运行过程中检测到了不应出现的异常时，在framework中通常是因为CHECK宏导致，不满足条件则触发fatal。该类型也比较好定位，因为是主动报错所以会打印出代码中的上下文信息，如上图所示，对应的代码文件路径、出错函数、CHECK条件等都已经打印出来。　

## 其它类型   ##
Signal 9——SIGKILL
进程被Kill，可能是来源于进程自身，也可能是来源于其它进程或者系统，比如低内存情况下清理系统后台应用时，还有应用发生ANR然后用户选择结束进程等，相对少见。

Signal 4——SIGILL
非法指令，可能原因是程序的二进制文件损坏或者内存跳变等原因，在Seattle平台早期上出现过不少次，目前在产线已经有了拦截方案，所以几乎没有见到了。


确认发生问题的位置

通过signal推测出大致出错原因后，下一步就是找到具体出错的文字。留意日志中的调用堆栈，找到最近且熟悉的模块（一般libc这种标准库不会出错），比如途中就是libstaefrght.so，然后在服务器上执行下列命令（必须先初始化编译环境）：

    addr2line -a 地址 -e 含有符号表的库文件

对应上图所示日志，以DAVINCE为例完整命令行如下：

    addr2line -a 00093e7b -e ./out/target/product/DAVINCE/symbols/system/lib/libstagefright.so

然后，就会打印出对应的代码文件和行数（因本地代码和正式版本编译代码的差异，多少存在误差）


> 通过signal和addr2line命令，已经能处理大部分tombstone类问题，但对于某些报错报到libc或者libart中，又或者位于系统关键服务中，出错可能涉及到多线程环境以及上下文处理流程等复杂情况，无法简单处理，就需要根据报错时的具体情况来分析，此时就要分析adb或者内核日志，还原当时的场景以确认原因。出错时间的确认也很简单，打开adb日志，以“fatal”作为关键字搜索，找到上图中类似的日志，然后对照tombstone的日志对比就可以确认具体出错时间，然后向上分析场景。



# Application ANR #

> ANR全称Application Not Responding，通常是一段时间内应用没有响应系统调用导致，默认情况下，Activity的最长执行时间是5秒，BroadcastReceiver的最长执行时间则是10秒，超过这个时间系统就会报ANR，弹出对话框问用户是继续等待应用响应还是强制结束应用进程。造成ANR的原因有很多，如主线程执行耗时操作、死锁、条件等待不满足、同步跨进程调用未返回等。  

该类APR日志主要看system_app_anr@xxxxxxxx.txt.gz或者trace.txt日志其中之一（记录信息基本相同，主要是ANR进程内所有线程的堆栈）。

## ANR——主线程执行高耗时操作 ##

![](https://i.imgur.com/wMUIuoA.png)

ANR最简单的一种情况，主线程执行了耗时操作，系统指定时间内未返回触发ANR。解决方案也很多：
设计上保证主线程中不会执行过于耗时的操作
优化代码尽量避免高耗时
将无法避免的耗时操作（如I/O、外部网络端数据的下载、大量的计算）移动到独立线程中（需要主要线程间的同步，以及考虑是否要采用进度条等UI给予用户提示）
处理过程发生了某种异常导致陷入死循环或者其它导致不能正常返回的情况

![](https://i.imgur.com/sSjyyq3.png)

打开对应的ANR或者trace.txt日志，分析主线程的调用堆栈，对于这类问题基本都可以明确的看到耗时操作，以上图中的测试程序为例，代码中在onCreate方法中有一个死循环，这样就必定会导致ANR，日志中主线程的堆栈信息中也明确打印了出了对应的代码位置，只要根据堆栈分析下代码一般都能找到原因，如寻找是不是可能出现死循环；是否卡在I/O操作中等等可能引起长时间执行而不返回的情况。

## ANR——条件等待不满足 ##

![](https://i.imgur.com/XM0bNOP.png)

本质上是前一种情况的变种，虽然将耗时操作移动到了独立线程中，但可能由于逻辑上的原因，必须等待这些耗时操作的完成（如下一步的计算以来前一步的计算结果等），解决方案依然同前述情况一致，只是这种情况下日志中主线程堆栈信息可能如下图所示，可以通过条件等待位置代码去分析在等待哪一部分工作完成，以及分析其它线程的堆栈来做进一步的判断。


## ANR——死锁 ##

![](https://i.imgur.com/jZWNTQc.png)

作为多线程的环境的顽疾，可以轻松的导致程序假死，对于应用来说就是ANR，但是好在死锁基本上都会完美的反应到对应的ANR日志中，分析起来也不算太困难，困难的主要是在复杂的流程中如何去解决（如Android的关键服务WindowManagerService、ActivityManagerService）。

死锁类问题都可以明显看到被阻塞(Blocked)的线程，其中需要等待的锁，以及持有这个锁的线程id都会明确的打印出来。

![](https://i.imgur.com/gW0jOC7.png)

![](https://i.imgur.com/aK870up.png)

主线程的堆栈中我们可以明确看到是91号线程持有了主线程想申请的锁，我们可以直接去看对应线程的堆栈，可以发现其又在申请23号线程持有的一个锁，于是我们要向上分析23号线程的堆栈（需要注意日志中会将多个系统多个在运行的进程堆栈都打印出来，所以23号线程可能不只一个，我们需要看的是同一进程下的23号线程）。

![](https://i.imgur.com/a6f1fg5.png)

然而23号线程又是在申请90号线程持有的锁，还得进一步分析。

![](https://i.imgur.com/rlgSMUD.png)

　分析至此，已可以确定，就是23号线程和90号线程互相申请对方持有的锁导致死锁，而23号线程之前所持有的锁又影响到91号线程，最终影响到主线程引发ANR。虽然已经确定了ANR的原因，但是这只是开始，剩下的就是根据堆栈中的信息分析对应的代码，找出引发死锁的原因并解决。


## 容易造成误会的条件等待 ##

![](https://i.imgur.com/ztIpxAS.png)

需要注意的是像上图这种堆栈，看起来像是犯了低级错误的自锁自申请式死锁，但实际上这种情况是条件等待或者Object的wait方法的正常使用方式，但是堆栈打印后看起来很容易引起误会。该wait方法要求先持有对应的锁，然后再调用wait方法，进入wait状态后系统会释放持有的锁，直到收到对应的notify通知或者signal信号，然后才会再次持有该锁，如锁已被其它线程持有，则等待，反应带堆栈中就是上图中的情况。


## 条件等待的用法 ##

![](https://i.imgur.com/0zsbQgA.png)

上图是条件等待（Condition）的常见用法——先持锁，然后进入wait状态并释放，直到收到signal，然后再持有锁，处理完毕后释放。
上图则是另外一种建议用法，先用关键字synchronized 持有对象的锁，然后调用对象的wait方法，等待对应的notify通知。

ANR——跨进程调用

![](https://i.imgur.com/K8e40CF.png)

跨进程调用引起的ANR很类似于最先介绍的“主线程执行高耗时操作”，只是耗时操作从自身进程换到了其它进程中，可能由于服务进程中执行的操作就很耗时、或者服务进程繁忙来不及处理又或是服务进程自身陷入了死锁等情况导致同步跨进程调用一直未能返回，引发ANR。
　　原理上虽然很简单，但是因为是跨进程调用，导致日志分析起来较为困难，看对应的trace日志都是到了Binder，然后就觉得无从下手。


![](https://i.imgur.com/QsycelM.png)

对于这类ANR问题，查看trace日志主线程的通常会看到在应用层的调用之后紧跟一个libbinder.so，然后堆栈是都是清一色的binder调用，在不清楚binder跨进程调用机制的情况下难以进行下一步的分析。
　　binder本质上只是google为了方便实现进程间通信而实现的一种机制，由，可以较为方便的让程序员实现与服务端的互相访问，遇到这类型的ANR，需要找对binder对应的服务端，去分析服务端为何长时间不返回。


![](https://i.imgur.com/1lxMKhW.png)

应用与服务端在跨进程调用时的关系可以简要概括为如上图，服务端会给予BnInterface派生一个类，其负责调用实际的处理过程，同时还会给予BpInterface派生一个类，其作为代理，负责通过内核跨进程调用，即跨进程调用BnInterface派生类的同名方法（也可不同名，但为了维护及代码阅读方便通常会使用同样的方法名称），这个代理类则提供给应用使用，所以遇到堆栈中看到Binder类的跨进程调用堆栈，下一步就是找对应的服务端BnInterface实现，而这依赖于对应服务模块的熟悉程度。




	"Binder_1" sysTid=2422
	  #00  pc 000182e4  /system/lib/libc.so (__futex_syscall3+8)
	  #01  pc 0000e5fc  /system/lib/libc.so (__pthread_cond_timedwait_relative+48)
	  #02  pc 0000e658  /system/lib/libc.so (__pthread_cond_timedwait+60)
	  #03  pc 0000e6f0  /system/lib/libc.so (pthread_join+108)
	  #04  pc 00099907  /system/lib/libstagefright.so (android::LowPowerPlayer::requestAndWaitHiFiReqCmdThreadExit()+14)
	  #05  pc 0009a7dd  /system/lib/libstagefright.so (android::LowPowerPlayer::~LowPowerPlayer()+52)
	  #06  pc 0009aa91  /system/lib/libstagefright.so (android::LowPowerPlayer::~LowPowerPlayer()+4)
	  #07  pc 00061aff  /system/lib/libstagefright.so (android::AwesomePlayer::reset_l()+318)
	  #08  pc 00061cc3  /system/lib/libstagefright.so (android::AwesomePlayer::reset()+146)
	  #09  pc 0003075f  /system/lib/libmediaplayerservice.so (android::StagefrightPlayer::reset()+4)
	  #10  pc 0002c561  /system/lib/libmediaplayerservice.so (android::MediaPlayerService::Client::reset()+28)
	  #11  pc 0004ca31  /system/lib/libmedia.so (android::BnMediaPlayer::onTransact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+536)
	 #12  pc 0001439d  /system/lib/libbinder.so (android::BBinder::transact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+60)
	  #13  pc 00016f99  /system/lib/libbinder.so (android::IPCThreadState::executeCommand(int)+516) 


　　简易快速分析方法——利用一般情况下服务端的Binder派生类方法名和代理类的方法名一致这一点，直接在服务端进程去搜同类名的方法，例如先前的例子中是MediaPlayer::reset()方法，那么直接搜索reset()方法，即可搜索到到对应的调用堆栈，当然有可能不只一处，所以搜到后还需要继续看代码确认下是否对应的服务端实现，确认后就和一般的ANR问题分析解决方法一致了。具体的例子及详细分析方法可参考右上角的文档。




# VMReboot #

> VMReboot本质上就是系统关键服务（ActivityManagerService、WindowManagerService、PowerManagerService等）或者进程（system_server、surfaceflinger等）出现了Crash或者Tombstone，此时系统就会重启虚拟机以将系统还原成初始状态。处理方式本质上和Crash和Tombstone处理方法一样，分析对应的日志找到出错原因并修正。  

Watchdog类问题本质上还是ANR，但是区别在于ANR一般是引用自身失去响应，而Watchdog则是各类系统重要服务失去响应（如ActivityManagerService、WindowManagerService等），然后挂载这些服务的进程system_server被杀后，必然会导致虚拟机重启。由于本质上相同，所以分析的方法也同ANR一致，重点看system_server_watchdog@xx日志，其和trace日志结构差不多，结合对应时间点的adb分析失去响应的原因并解决即可，只不过由于都是系统关键服务，代码复杂度较高，分析起来相对比较困难。

死机重启问题请见另一篇blog
https://www.wyongch.top/2018/05/15/Android%E6%AD%BB%E6%9C%BA%E9%87%8D%E5%90%AF%E9%97%AE%E9%A2%98%E5%88%86%E6%9E%90%E6%80%9D%E8%B7%AF/
