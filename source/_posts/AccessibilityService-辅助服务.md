---
title: AccessibilityService 辅助服务
date: 2018-06-14 09:18:01
tags:
- Android

---

AccessibilityService是什么，首先看一下官方解释：

> Accessibility services are intended to assist users with disabilities in using Android devices and apps. They run in the background and receive callbacks by the system whenAccessibilityEvents are fired.

无障碍服务旨在帮助残疾人使用Android设备和应用程序。它们在后台运行，当系统被触发时接收系统的回调。

<!--more-->

AccessibilityService的出现是为了帮助使用不便的人去使用Android设备和应用，但是当你了解这个服务之后，他的功能远不止如此。
AccessibilityService可以拦截到系统发出的一些消息（比如窗体状态的改变，通知栏状态的改变，View被点击了等等），
当拦截到这些事件我们就可以去做一些我们想做的事,功能和强大
AccessibilityService具体能做些什么呢？ 比如自动化测试、自动抢红包、自动安装等等。

# AccessibilityService 创建 #

	public class MyAccessibilityService extends AccessibilityService {
    @Override
    public void onAccessibilityEvent(AccessibilityEvent accessibilityEvent) {

    }

    @Override
    public void onInterrupt() {

    }
}

## 创建AccessibilityService配置文件 ##

在res/xml目录下新建一个xml文件accessibility_config.xml（文件名随意）,后面会对这个文件作详细的介绍。


	<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:accessibilityEventTypes="typeAllMask"
    android:accessibilityFeedbackType="feedbackGeneric"  
    android:canRetrieveWindowContent="true"
    android:description="@string/accessibility_config"
    android:notificationTimeout="100"
    android:packageNames="com.tencent.mobileqq,com.android.packageinstaller" />

## 注册AccessibilityService 以及添加权限##

在AndroidManifest.xml中注册AccessibilityService 以及添加权限

	 <service
            android:name=".service.MyAccessibilityService"
            android:enabled="true"
            android:exported="true"
            android:label="@string/app_name"
            android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
            <intent-filter>
                <action android:name="android.accessibilityservice.AccessibilityService" />
            </intent-filter>
            <meta-data
                android:name="android.accessibilityservice"
                android:resource="@xml/accessibility_config" />//配置文件accessibility_config.xml
        </service>

添加`BIND_ACCESSIBILITY_SERVICE`权限

	<uses-permission android:name="android.permission.BIND_ACCESSIBILITY_SERVICE" />

如上：AccessibilityService服务基本环境就配置好了，可以在手机中绑定服务并且运行。

# 使用AccessibilityService #

配置文件xml文件中各个属性的含义吧

<code>accessibilityEventTypes </code>： 用来设置响应事件的类型，比如typeAllMask就是响应全部事件，
typeNotificationStateChanged就是响应通知状态的改变，如果需要响应多种事件类型可以以 ‘ | ’ 隔开。  
<code>accessibilityFeedbackType </code>： 给用户的反馈方式，比如语音、震动等，这里用处不大。  
<code>canRetrieveWindowContent</code> ：是否可以获取活动窗体的内容，这个设置为true才可以取得窗体中的控件和事件源  
<code>description</code> ：辅助功能的描述  
<code>notificationTimeout</code> ：两个相同类型事件发送到服务的事件间隔，单位毫秒  
<code>packageNames</code> ：指定响应某个应用的事件，取值为应用的包名，多个以‘ , ’ 隔开。没有此属性则表示响应全部应用。这里我填写的是手机qq和系统安装器的包名  

然后进入到我们的<code>MyAccessibilityService </code>中，定位到<code>onAccessibilityEvent</code>方法编写如下代码

	log("-------------------------------------------------------------");
        int eventType = event.getEventType();//事件类型
        log("packageName:" + event.getPackageName() + "");//响应事件的包名，也就是哪个应用才响应了这个事件
        log("source:" + event.getSource() + "");//事件源信息
        log("source class:" + event.getClassName() + "");//事件源的类名，比如android.widget.TextView
        log("event type(int):" + eventType + "");

        switch (eventType) {
            case AccessibilityEvent.TYPE_NOTIFICATION_STATE_CHANGED:// 通知栏事件
                log("event type:TYPE_NOTIFICATION_STATE_CHANGED");
                break;
            case AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED://窗体状态改变
                log("event type:TYPE_WINDOW_STATE_CHANGED");
                break;
            case AccessibilityEvent.TYPE_VIEW_ACCESSIBILITY_FOCUSED://View获取到焦点
                log("event type:TYPE_VIEW_ACCESSIBILITY_FOCUSED");
                break;
            case AccessibilityEvent.TYPE_GESTURE_DETECTION_START:
                log("event type:TYPE_VIEW_ACCESSIBILITY_FOCUSED");
                break;
            case AccessibilityEvent.TYPE_GESTURE_DETECTION_END:
                log("event type:TYPE_GESTURE_DETECTION_END");
                break;
            case AccessibilityEvent.TYPE_WINDOW_CONTENT_CHANGED:
                log("event type:TYPE_WINDOW_CONTENT_CHANGED");
                break;
            case AccessibilityEvent.TYPE_VIEW_CLICKED:
                log("event type:TYPE_VIEW_CLICKED");
                break;
            case AccessibilityEvent.TYPE_VIEW_TEXT_CHANGED:
                log("event type:TYPE_VIEW_TEXT_CHANGED");
                break;
            case AccessibilityEvent.TYPE_VIEW_SCROLLED:
                log("event type:TYPE_VIEW_SCROLLED");
                break;
            case AccessibilityEvent.TYPE_VIEW_TEXT_SELECTION_CHANGED:
                log("event type:TYPE_VIEW_TEXT_SELECTION_CHANGED");
                break;
        }

        for (CharSequence txt : event.getText()) {
            log("text:" + txt);//输出当前事件包含的文本信息
        }

        log("-------------------------------------------------------------");

log方法就是打印信息用的，是对Log的一个简单封装。

在运行一次程序，打开我们的手机qq可以看见输出日志如下


可以很清晰的看见当我们打开qq或触发很多的事件，第一个响应的事件是<code>TYPE_WINDOW_STATE_CHANGED</code>,触发此事件的事件源是<code>com.tencent.mobileqq.activity.SplashActivity</code>

接着，当我点击最上面的 ‘ 电话 ’，会得到如下日志：


我就不一一截图操作的日志了，下来大家可以自行尝试。到这里你用该对<code>onAccessibilityEvent</code>有了进一步的了解。

# 构建一个apk自动安装器 #

这一节来个实战的应用：做一个apk的自动安装器，点击apk文件即可开始自动安装。类似于应用宝中的省心安装功能。


正常情况点击安装包之后会弹出一个确认框，点击继续之后会出现下一步，点击了下一步就可以点击安装了。我们要实现自动安装无非就是用程序的方式会对相关按钮的自动点击，难点就在于怎么去找到这些按钮并执行点击操作。

好了，下面我们可以来创建一个自动安装的服务了，步骤和第一节描述的一样将<code>packageNames</code>指定成<code>com.android.packageinstaller</code>就可以了。
这里我们先别着急去实现功能（就算你想去实现，也摸不着头脑），我们还是像第二节那样去输出日志信息，查看日志输出信息（对响应事件信息的打印）是使用AccessibilityService的重点，看完日志的输出信息，然后对其分析，才会知道具体的操作。
我们来看看在安装apk文件时候输出的部分日志信息：




从日志信息中可以看出：  
1.当我们点击了apk文件的时候，如果该文件已经存在，就会弹出一个对话框并且会响应<code>TYPE_WINDOW_STATE_CHANGE</code>事件  
2.当我们点击了继续之后就会响应<code>TYPE_VIEW_CLICKED</code>事件，并且继续这个可以点击的View是个Button  
3.接着点击下一步，同样也会响应<code>TYPE_VIEW_CLICKED</code>事件，并且继续这个可以点击的View是个TextView。最后就可以点击安装了（安装那步忘了截图，和第3步相应的事件是一样的）

经过上面几步的分析，我想你应该对后面的逻辑还是比较清楚了，直接上代码


	public class AutoInstallService extends AccessibilityService {

    @Override
    public void onAccessibilityEvent(AccessibilityEvent event) {
        PrintUtils.printEvent(event);
        findAndPerformActionButton("继续");
        findAndPerformActionTextView("下一步");
        findAndPerformActionTextView("安装");
    }


    private void findAndPerformActionButton(String text) {
        if (getRootInActiveWindow() == null)//取得当前激活窗体的根节点
            return;
        //通过文字找到当前的节点
        List<AccessibilityNodeInfo> nodes = getRootInActiveWindow().findAccessibilityNodeInfosByText(text);
        for (int i = 0; i < nodes.size(); i++) {
            AccessibilityNodeInfo node = nodes.get(i);
            // 执行点击行为
            if (node.getClassName().equals("android.widget.Button") && node.isEnabled()) {
                node.performAction(AccessibilityNodeInfo.ACTION_CLICK);
            }
        }
    }

    private void findAndPerformActionTextView(String text) {
        if (getRootInActiveWindow() == null)
            return;
        //通过文字找到当前的节点
        List<AccessibilityNodeInfo> nodes = getRootInActiveWindow().findAccessibilityNodeInfosByText(text);
        for (int i = 0; i < nodes.size(); i++) {
            AccessibilityNodeInfo node = nodes.get(i);
            // 执行按钮点击行为
            if (node.getClassName().equals("android.widget.TextView") && node.isEnabled()) {
                node.performAction(AccessibilityNodeInfo.ACTION_CLICK);
            }
        }
    }
}

例子很简单，所以也没去对eventType做判断，当有其他需求的时候是需要对eventType做判断了，代码已经注释的比较详细，我就不再对代码作更多的讲解了


补充说明
AccessibilityService中还有几个常用的方法 onServiceConnected、onInterrupt、onGesture。看名字大致也知道什么时候会被调用。

在onServiceConnected中可以去配置AccessibilityService的一些信息，也就是之前在xml文件可以在这里通过代码配置，不过这里没法配置canRetrieveWindowContent属性，刚开始也被这个坑了很久

	@Override
    protected void onServiceConnected() {
        super.onServiceConnected();
        PrintUtils.log("onServiceConnected");
    //        //可用代码配置当前Service的信息
    //        AccessibilityServiceInfo info = new AccessibilityServiceInfo();
    //        info.packageNames = new String[]{"com.android.packageinstaller", "com.tencent.mobileqq", "com.trs.gygdapp"}; //监听过滤的包名
    //        info.eventTypes = AccessibilityEvent.TYPES_ALL_MASK; //监听哪些行为
    //        info.feedbackType = AccessibilityServiceInfo.FEEDBACK_SPOKEN; //反馈
    //        info.notificationTimeout = 100; //通知的时间
    //        setServiceInfo(info);
    }



