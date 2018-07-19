---
title: xUtile 框架分析
date: 2018-06-07 19:40:01
tags:
- xUtile

---


在分析xUtile 框架之前我一直想着如何用简单的语言把复杂的逻辑讲清楚，今天将跟大家一起从宏观结构和具体实现细节两个方面把这个框架捋一捋

<!--more-->

xUtile 主要分为四个模块 view image http db

初始化时序图如下：

![](https://i.imgur.com/F5FvMlv.png)

view 模块

view 主要涉及的代码如下：  
注解类
D:\AndroidStudioProjects\xUtils3\xutils\src\main\java\org\xutils\view\annotation\ContentView.java
D:\AndroidStudioProjects\xUtils3\xutils\src\main\java\org\xutils\view\annotation\Event.java
D:\AndroidStudioProjects\xUtils3\xutils\src\main\java\org\xutils\view\annotation\ViewInject.java view事件监听管理者
D:\AndroidStudioProjects\xUtils3\xutils\src\main\java\org\xutils\view\EventListenerManager.java 根据id查找对应的view
D:\AndroidStudioProjects\xUtils3\xutils\src\main\java\org\xutils\view\ViewFinder.java 定义viewinfo对象，重写equals 和hashcode方法，主要用于判断两个view是否相等
D:\AndroidStudioProjects\xUtils3\xutils\src\main\java\org\xutils\view\ViewInfo.java view 注入具体实现
D:\AndroidStudioProjects\xUtils3\xutils\src\main\java\org\xutils\view\ViewInjectorImpl.java view 注入接口
D:\AndroidStudioProjects\xUtils3\xutils\src\main\java\org\xutils\ViewInjector.java


view 模块主要的功能是 

1. 设置Activity ， Fragment 对应的 contentView，setContentView()
2. 初始化 Activity ，Fragment 中 view 控件，一般是 findViewById()
3. 对应view 控件的事件绑定 

因此 框架中view 模块主要定义了三个注解类  ContentView， Event， ViewInject

view模块主要需要完成三件事情：

1. 加注解（开发者需要做的事情）  
2. 注解在代码中只是静态备注，因此需要将定义的view动态注入对应的控件中（框架实现）  
3. 对控件添加事件绑定（框架实现）

代码中使用方法如下，开发者只需要在对应的 class 或者 成员变量，成员方法上加上对应的标签就行，这只是已静态的形式添加标签而已，具体的功能实现是在 ViewInjectorImpl 中

	@ContentView(R.layout.activity_main)
	public class MainActivity extends BaseActivity {
	    @ViewInject(R.id.container)
	    private ViewPager mViewPager;
	
	    @ViewInject(R.id.toolbar)
	    private Toolbar toolbar;
	}
	
	
	@ContentView(R.layout.fragment_http)
	public class HttpFragment extends BaseFragment {
	    // 事件绑定，函数名称可以任意
	    @Event(value = R.id.btn_test1,
	            type = View.OnClickListener.class/*可选参数, 默认是View.OnClickListener.class*/)
	    private void onTest1Click(View view) {}
	}




view模块中最核心的就是 ViewInjectorImpl 是view注入的具体实现

从ViewInjector.java 中我们可以看出，主要需要实现 view，view hold，activity，fragment 控件的注入

    // 参数为传递过来的activity
    @Override
    public void inject(Activity activity) {
        //获取Activity的ContentView的注解，获取class
        Class<?> handlerType = activity.getClass();
        try {
            // 根据自定义函数findContentView 获取activity 的contentview注解
            ContentView contentView = findContentView(handlerType);
            if (contentView != null) {
                // 将注解value值，赋值给viewId
                int viewId = contentView.value();
                if (viewId > 0) {
                    // 根据反射获取class 的setContentView方法
                    Method setContentViewMethod = handlerType.getMethod("setContentView", int.class);
                    // invoke 执行setContentViewMethod 方法，参数为注解中定义的viewId
                    setContentViewMethod.invoke(activity, viewId);
                }
            }
        } catch (Throwable ex) {
            LogUtil.e(ex.getMessage(), ex);
        }
        //以上代码相当于执行了 setContentView(View view)

        injectObject(activity, handlerType, new ViewFinder(activity));
    }

获取contentview注解

    private static ContentView findContentView(Class<?> thisCls) {
        if (thisCls == null || IGNORED.contains(thisCls)) {
            return null;
        }
        ContentView contentView = thisCls.getAnnotation(ContentView.class);
        if (contentView == null) {
            //递归调用，如果子类contentview注解为空，则从父类获取
            return findContentView(thisCls.getSuperclass());
        }
        return contentView;
    }


如上代码执行serContentView 之后将会跳转到injectObject(activity, handlerType, new ViewFinder(activity));
injectObject 函数中主要分为如下两部分：

第一部分：主要是将 Activity中ViewInject注解的成员变量值设成 注解中配置的value值，即 （eg：mViewPager = findViewById(R.id.container)）

      // 从父类到子类递归
        // handler = MainActivity, handlerType.getSuperclass() = MainActivity.class.getSuperclass, finder
        injectObject(handler, handlerType.getSuperclass(), finder);

        // inject view
        // 通过反射获取所以 handlerType （eg：MainActivity.class ）所以参数数组
        Field[] fields = handlerType.getDeclaredFields();
        if (fields != null && fields.length > 0) {
            for (Field field : fields) {

                Class<?> fieldType = field.getType();
                if (
                /* 不注入静态字段 */     Modifier.isStatic(field.getModifiers()) ||
                /* 不注入final字段 */    Modifier.isFinal(field.getModifiers()) ||
                /* 不注入基本类型字段 */  fieldType.isPrimitive() ||
                /* 不注入数组类型字段 */  fieldType.isArray()) {
                    continue;
                }

                // 如果参数是 ViewInject注解
                ViewInject viewInject = field.getAnnotation(ViewInject.class);
                if (viewInject != null) {
                    try {
                        // 通过viewFinder 对象和 注解value 找到对应activity的view （viewInject.value()注解定义的值）
                        View view = finder.findViewById(viewInject.value(), viewInject.parentId());
                        if (view != null) {
                            field.setAccessible(true);
                            // 通过反射设置 handler （eg：MainActivity）类中 field （eg：private ViewPager mViewPager;）属性值为 view （eg：@ViewInject(R.id.container)）
                            field.set(handler, view);
                        } else {
                            throw new RuntimeException("Invalid @ViewInject for "
                                    + handlerType.getSimpleName() + "." + field.getName());
                        }
                    } catch (Throwable ex) {
                        LogUtil.e(ex.getMessage(), ex);
                    }
                }
            }
        } // end inject view

第二部分：将activity中对应的view控件注入事件绑定

通过递归调用注入所有 view ， 然后调用 EventListenerManager.addEventMethod(finder, info, event, handler, method); 添加事件绑定

先来看一下正常情况下设置事件监听方法如下：

        View.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //method()
            }
        });

由此我们可以看出，需要明确知道 view ， view上的setOnClickListener ， View.OnClickListener() 对象 ， onClick 回调函数 ，method 方法

如下代码的目的就是需要将 method 方法添加到 View.OnClickListener() 对象的 onClick 方法中

     public static void addEventMethod(
            //根据页面或view holder生成的ViewFinder
            ViewFinder finder,
            //根据当前注解ID生成的ViewInfo
            ViewInfo info,
            //注解对象
            Event event,
            //页面或view holder对象
            Object handler,
            //当前注解方法
            Method method) {
        try {
            // 根据viewInfo获取需要绑定的view对象
            View view = finder.findViewByInfo(info);

            if (view != null) {
                // 注解中定义的接口，比如Event注解默认的接口为View.OnClickListener
                Class<?> listenerType = event.type();
                // 默认为空，注解接口对应的Set方法，比如setOnClickListener方法
                String listenerSetter = event.setter();
                if (TextUtils.isEmpty(listenerSetter)) {
                    // listenerType = View.OnClickListener.class
                    // listenerSetter = "setOnClickListener"
                    listenerSetter = "set" + listenerType.getSimpleName();
                }


                // event.method 默认为空
                String methodName = event.method();

                boolean addNewMethod = false;
                /*
                    根据View的ID和当前的接口类型获取已经缓存的接口实例对象，
                    比如根据View.id和View.OnClickListener.class两个键获取这个View的OnClickListener 缓存对象
                 */
                Object listener = listenerCache.get(info, listenerType);
                DynamicHandler dynamicHandler = null;
                /*
                    如果接口实例对象不为空
                    获取接口对象对应的动态代理对象
                    如果动态代理对象的handler和当前handler相同
                    则为动态代理对象添加代理方法
                 */
                if (listener != null) {
                    // 根据动态代理获取OnClickListener 的代理对象
                    dynamicHandler = (DynamicHandler) Proxy.getInvocationHandler(listener);
                    addNewMethod = handler.equals(dynamicHandler.getHandler());
                    if (addNewMethod) {
                        // eg: methodName = null ,method = onTest1Click(View view)
                        dynamicHandler.addMethod(methodName, method);
                    }
                }

                // 如果还没有注册此代理
                if (!addNewMethod) {

                    //新建代理对象，eg: handler = HttpFragment
                    dynamicHandler = new DynamicHandler(handler);

                    dynamicHandler.addMethod(methodName, method);

                    // 生成的代理对象实例，比如View.OnClickListener的实例对象
                    listener = Proxy.newProxyInstance(
                            //eg: listenerType = View.OnClickListener
                            listenerType.getClassLoader(),
                            new Class<?>[]{listenerType},
                            dynamicHandler);

                    listenerCache.put(info, listenerType, listener);
                }

                // 获取view 的onClickListener 相当于 View.setOnClickLisner()
                Method setEventListenerMethod = view.getClass().getMethod(listenerSetter, listenerType);

                setEventListenerMethod.invoke(view, listener);
                Log.i("EventListenerManager", "wyc addEventMethod view="+view+"  listener="+listener+"  listenerSetter="+listenerSetter+"  listenerType="+listenerType+"  dynamicHandler="+dynamicHandler+"  handler="+handler);
                // I/EventListenerManager: wyc addEventMethod view=android.support.v7.widget.AppCompatButton{a584485 VFED..C.. ......I. 0,0-0,0 #7f0d008e app:id/btn_test1}  listener=DynamicHandler  listenerSetter=setOnClickListener  listenerType=interface android.view.View$OnClickListener  dynamicHandler=org.xutils.view.EventListenerManager$DynamicHandler@fb95b1c  handler=HttpFragment{6f1eba2 #0 id=0x7f0d0079 android:switcher:2131558521:0}

            }
        } catch (Throwable ex) {
            LogUtil.e(ex.getMessage(), ex);
        }
    }


动态代理的作用是执行onClick 中method方法 eg:onTestDbClick() 

     public static class DynamicHandler implements InvocationHandler {
        // 存放代理对象，比如Fragment或view holder
        private WeakReference<Object> handlerRef;
        // 存放代理方法
        private final HashMap<String, Method> methodMap = new HashMap<String, Method>(1);

        private static long lastClickTime = 0;

        public DynamicHandler(Object handler) {
            this.handlerRef = new WeakReference<Object>(handler); // eg: handler=HttpFragment
        }

        public void addMethod(String name, Method method) {
            methodMap.put(name, method);
        }

        public Object getHandler() {
            return handlerRef.get();
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            Object handler = handlerRef.get();
            if (handler != null) {

                String eventMethod = method.getName();
                if ("toString".equals(eventMethod)) {
                    return DynamicHandler.class.getSimpleName();
                }

                method = methodMap.get(eventMethod);
                if (method == null && methodMap.size() == 1) {
                    for (Map.Entry<String, Method> entry : methodMap.entrySet()) {
                        if (TextUtils.isEmpty(entry.getKey())) {
                            method = entry.getValue();
                        }
                        break;
                    }
                }

                if (method != null) {

                    if (AVOID_QUICK_EVENT_SET.contains(eventMethod)) {
                        long timeSpan = System.currentTimeMillis() - lastClickTime;
                        if (timeSpan < QUICK_EVENT_TIME_SPAN) {
                            LogUtil.d("onClick cancelled: " + timeSpan);
                            return null;
                        }
                        lastClickTime = System.currentTimeMillis();
                    }

                    try {
                        Log.i("EventListenerManager", "wyc invoke hander="+handler+ "  args="+args+" method="+method+" eventMethod="+eventMethod+" methodMap="+methodMap);
                        // I/EventListenerManager: wyc invoke hander=DbFragment{d2ad05a #1 id=0x7f0d0079 android:switcher:2131558521:1}  args=[Ljava.lang.Object;@f2693bb method=private void org.xutils.sample.DbFragment.onTestDbClick(android.view.View) eventMethod=onClick methodMap={=private void org.xutils.sample.DbFragment.onTestDbClick(android.view.View)}
                        return method.invoke(handler, args);
                    } catch (Throwable ex) {
                        throw new RuntimeException("invoke method error:" +
                                handler.getClass().getName() + "#" + method.getName(), ex);
                    }
                } else {
                    LogUtil.w("method not impl: " + eventMethod + "(" + handler.getClass().getSimpleName() + ")");
                }
            }
            return null;
        }
    }




