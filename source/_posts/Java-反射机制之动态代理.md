---
title: Java 反射机制之动态代理
date: 2018-06-27 17:08:42
tags:
- Java

---





Java 反射机制中有一个非常常用的设计模式——动态代理，通过动态代理，我们可以非常方便的访问真实对象中的接口

<!--more-->


下面将具体说明这一设计模式


1. Subject 抽象对象
2. RealSubject 实际执行对象
3. DynamicProxy 动态代理对象


Subject 抽象类，主要定义相关接口

	package com.wyongch;

	public interface Subject {
		
		public void hello(String str);
		
		public void rent ();
	
	}

RealSubject 实际执行对象，需要实现Subject 抽象接口

	package com.wyongch;
		
	public class RealSubject implements Subject {
	
		public void hello(String str) {
			// TODO Auto-generated method stub
			System.out.print("on excute RealSubject hello "+str+ "\n");
	
		}
	
		public void rent() {
			// TODO Auto-generated method stub
			System.out.print("on excute RealSubject rent \n");
	
		}
	
	}

DynamicProxy 动态代理类，需要实现 InvocationHandler 接口

	package com.wyongch;
	
	import java.lang.reflect.InvocationHandler;
	import java.lang.reflect.Method;
	
	public class DynamicProxy implements InvocationHandler {
		
		private Object subject ;
		
		public DynamicProxy(Object obj){
			this.subject = obj ;
		}
		
	
		public Object invoke(Object proxy, Method method, Object[] args)
				throws Throwable {
			// TODO Auto-generated method stub
			
			System.out.print(">>>>>>>>>>> DynamicProxy invoke \n");
			
			method.invoke(subject, args);
			
			System.out.print("<<<<<<<<<<<< DynamicProxy invoke \n");
			
			return null;
		}
	
	}

Client 端测试代码

	package com.wyongch;
	
	import java.lang.reflect.InvocationHandler;
	import java.lang.reflect.Proxy;
	
	public class Main {
	
		/**
		 * @param args
		 */
		public static void main(String[] args) {
			// TODO Auto-generated method stub
			System.out.print("................main............... \n");
			
			Subject subject = new RealSubject();
			
			InvocationHandler handler = new DynamicProxy(subject);
			
			
			Subject proxy = (Subject)Proxy.newProxyInstance(subject.getClass().getClassLoader(), subject.getClass().getInterfaces(), handler);
			
			System.out.print("  subject = handler "+subject.getClass().getClassLoader().equals(handler.getClass().getClassLoader()) +"\n ");
			
			System.out.print("handler.getClassLoader = "+handler.getClass().getClassLoader() +"  subject.getClassLoader = "+subject.getClass().getClassLoader()+ "\n");
			
			System.out.print("handler.getClass = "+handler.getClass() +"  subject.getClass = "+subject.getClass()+ "\n");
			
			System.out.print("subject="+subject+  "  handler = "+handler + "\n"); 
			
			proxy.rent();
			
			proxy.hello("wuyongchao");
			
			System.out.print(" >>handler.getClass = "+handler.getClass().getClassLoader() +"  subject.getClass = "+subject.getClass().getClassLoader()+ "\n");
	
		}
	
	}


执行结果

	................main............... 
	  subject = handler true
	 handler.getClassLoader = sun.misc.Launcher$AppClassLoader@73d16e93  subject.getClassLoader = sun.misc.Launcher$AppClassLoader@73d16e93
	handler.getClass = class com.wyongch.DynamicProxy  subject.getClass = class com.wyongch.RealSubject
	subject=com.wyongch.RealSubject@6bc7c054  handler = com.wyongch.DynamicProxy@232204a1
	>>>>>>>>>>> DynamicProxy invoke 
	on excute RealSubject rent 
	<<<<<<<<<<<< DynamicProxy invoke 
	>>>>>>>>>>> DynamicProxy invoke 
	on excute RealSubject hello wuyongchao
	<<<<<<<<<<<< DynamicProxy invoke 
	 >>handler.getClass = sun.misc.Launcher$AppClassLoader@73d16e93  subject.getClass = sun.misc.Launcher$AppClassLoader@73d16e93
