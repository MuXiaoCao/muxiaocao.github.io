---
title: 设计模式之代理模式（二）CGLIB动态代理实现
date:  2016-05-16
categories:  系统架构设计
keywords:  [设计模式， 动态代理，cglib]
tags: 
	- 设计模式
	- 动态代理
	- cglib
---
>像上一篇所说的代理模式其实是**静态代理**，在实际开发中其实应用不大，因为他**需要事先知道被代理对象是谁**，而且**被代理对象和代理对象实现了公共的接口**。实际情况往往并不能满足这些条件，我们往往在写代理模式的时候并不知道到时候被代理的对象是谁。解决办法就是——**动态代理**。以下我们将使用**CGLIB实现动态代理**。
## 一、动态代理概述
>**程序在运行期而不是编译器，生成被代理对象的代理对象，并且被代理对象并不需要和代理对象实现共同的接口**。基于此，我们可以利用代理对象，**提供一种以控制对被代理对象的访问**。
### 1.1 动态代理的原理
>基于CGLIB的动态代理，**其实就是利用了CGLIB，在运行时期生成被代理对象的子类，来作为代理对象**。同时重写了被代理对象的所有方法。当我们用代理对象调用方法的时候，其实是调用的重写后的方法。该方法实际调用的是callback中的用户自定义的代码逻辑，例如权限验证。**当验证通过了，则通过反射，反射出父类（被代理对象）的方法并调用（invoke）**。
>那么问题来了，**如何生成被代理对象的子类**呢？我们事先并不知道谁是代理对象啊？而且我们在重写调用自定义逻辑呢？
### 1.2 CGLIB动态代理的底层实现
>CGLIB是一个强大的高性能的代码生成包。
      1. 它广泛的被许多AOP的框架使用，例如：Spring AOP和dynaop，为他们提供方法的interception（拦截）；
       2. hibernate使用CGLIB来代理单端single-ended(多对一和一对一)关联（对集合的延迟抓取，是采用其他机制实现的）；
       3. EasyMock和jMock是通过使用模仿（moke）对象来测试Java代码的包。
     它们都通过使用CGLIB来为那些没有接口的类创建模仿（moke）对象。

>CGLIB包的底层是通过使用一个小而快的**字节码处理框架ASM(Java字节码操控框架)**，来转换字节码并生成新的类。他其实就像JVM一样，可以加载一个指定的类。这样我们就可以实现在运行时期生成我们自定义的class了。下图为**cglib与一些框架和语言的关系（CGLIB Library and ASM Bytecode Framework）**
![CGLIB结构](http://hi.csdn.net/attachment/201109/29/0_1317264486lqGr.gif)
## 二、应用场景
>1. 远程代理：也就是为一个对象在不同的地址空间提供局部代表。这样可以隐藏一个对象存在不同地址空间的事实。例如：WebService、RPC、RMI(Remote Method Invocation)等。
>2. 虚拟代理：是根据需要创建开销很大的对象。通过它来存放实例化需要很长时间的真实对象。例如：利用虚拟代理来优化页面的打开速度。
3. 安全代理：用来控制真实对象访问时的权限。一般用户对象应该在不同访问权限的时候。
4. 智能指引：是指当调用真实的对象时，代理处理另外一些事情。例如：Spring AOP、Hibernate等。
## 三、结构图
![这里写图片描述](http://img.blog.csdn.net/20160516164615927)
## 四、实际例子
### 4.1 需求：为自定义的BookServiceBean动态生成代理对象，实现权限管理，只允许姓名为张三的对象可以操作。
### 4.2 代码示例
**被代理对象**
```
package com.xiaocao.proxy.cglib;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class BookServiceBean {
	
	private Logger log = LogManager.getLogger();
	public BookServiceBean() {
		log.info("this is bookservicebean 的构造方法");
	}
	public void create() {
		System.out.println("create() is running!");
	}
	public void query() {
		System.out.println("query() is running!");
	}
}
```
**代理类**
```
package com.xiaocao.proxy.cglib;

import java.lang.reflect.Method;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

public class MyCglibProxy implements MethodInterceptor{

	private Logger log = LogManager.getLogger(MyCglibProxy.class);
	
	private String name;
	
	public MyCglibProxy(String name) {
		this.name = name;
	}
	
	public Object intercept(Object object, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {

		log.info("调用的方法是：" + method.getName());
		log.info("实际调用者是： " + object.getClass());
		for (Object obj : objects) {
			log.info("方法参数类型为：" + obj.getClass());
		}
		
		if (!name.equals("张三")) {
			System.out.println("权限不够");
			return null;
		}
		Object result = methodProxy.invokeSuper(object, objects);
		System.out.println("这是方法后");
		return result;
	}
}
```
**代理对象工厂**
```
package com.xiaocao.proxy.cglib;

import net.sf.cglib.proxy.Enhancer;

public class BookServiceFactory {
	
	private BookServiceFactory() {
		
	}
	public static BookServiceBean getProxyInstance(MyCglibProxy myProxy) {
		Enhancer enhancer = new Enhancer();
		// 将Enhancer中的superclass属性赋值成BookServiceBean
		enhancer.setSuperclass(BookServiceBean.class);
		// 将Enhancer中的callbacks属性赋值成myProxy
		enhancer.setCallback(myProxy);
		return (BookServiceBean) enhancer.create();
	}
}
```
**测试类**
>测试类的主题部分很简单，只是通过工厂方法得到代理对象，然后调用，即可通过姓名判断是否有权限执行目标方法。
>之后我用了许多反射机制来验证我之前想法，包括
>1. 得到的代理对象其实是被代理对象的子类
>2. 代理对象重写了被代理对象的所有方法，我们调用的时候其实是调用的被代理对象的重写方法

**具体代码如下：**
```
package com.xiaocao.proxy.cglib;

import java.io.File;
import java.io.FileOutputStream;
import java.lang.reflect.Field;
import java.lang.reflect.Method;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class Test {
	public static void main(String[] args) {
		Logger log = LogManager.getLogger();
		BookServiceBean service = BookServiceFactory.getProxyInstance(new MyCglibProxy("张三"));  
		service.create(); 

		log.info("我们得到的bean是：" + service.getClass());
		System.out.println("实际调用者的父类：" + service.getClass().getSuperclass());
		try {
			Class<?> c = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6");
			Class<?> beanc = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean");
			
			Method[] beanc_method = beanc.getMethods();
			int i = 1;
			System.out.println("原始的bean的方法总共" + beanc_method.length + "个");
			for (Method method : beanc_method) {
		
				System.out.println("原始的bean方法" + i++ + method.getName());
	
			}
			i = 1;
			Method[] methods = c.getMethods();
			System.out.println("我们得到的bean的方法总共" + methods.length + "个");
			for (Method method : methods) {
				System.out.println("我们得到的bean的方法" + i++ + method.getName());
			}
			System.out.println("原始的bean的父类：" + beanc.getSuperclass());
			System.out.println("我们得到的bean的父类：" + c.getSuperclass());

			Field[] bean_fields = beanc.getDeclaredFields();
			i = 1;
			for (Field field : bean_fields) {
				System.out.println("原始bean的属性 " + i++ + field);
			}
		
			Field[] fields = c.getDeclaredFields();
			i=1;
			for (Field field : fields) {
				System.out.println("我们得到的bean的属性 " + i++ + field);
			}
			Class proxyGenerator = Class.forName("sun.misc.ProxyGenerator");
			Method[] methods2 = proxyGenerator.getMethods();
			for (Method method : methods2) {
				System.out.println(method);
				byte[] TempProxySuper = (byte[]) method.invoke(proxyGenerator,"TempProxySuper", new Class[]{c.getSuperclass()});
				byte[] TempProxy = (byte[]) method.invoke(proxyGenerator,"TempProxy", new Class[]{c});
				byte[] TempBean = (byte[]) method.invoke(proxyGenerator,"TempBean", new Class[]{beanc});
				createClassFile("TempProxy",TempProxy);
				createClassFile("TempProxySuper",TempProxySuper);
				createClassFile("TempBean", TempBean);
				break;
			}
			
		} catch (ClassNotFoundException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} 
	}
	/**
	 * 生成class文件
	 * @param fileName
	 * @param classFile
	 */
	public static void createClassFile(String fileName,byte[] classFile) {
		try {
			File file;
			FileOutputStream fos = new FileOutputStream(file = new File(fileName+".class"));
		fos.write(classFile);
			fos.flush();
			fos.close();
	System.out.println(file.getAbsolutePath());
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}
}
```
**注意**：代码中的`com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6`，是我之前测试得到代理类名称。
### 4.3 测试结果
![这里写图片描述](http://img.blog.csdn.net/20160516171648890)
![这里写图片描述](http://img.blog.csdn.net/20160516173324053)

>测试代码的剩下部分是用来生成被代理类、代理类和代理类父类的class，其中用到了`sun.misc.ProxyGenerator`类的静态方法`proxyGenerator`,他需要的参数有
>1. 生成的类名
>2. 需要生成的class对象
>返回值则是`byte[]`是java的bytecode，可以用来生成class文件。之后就是调用`createClassFile`生成class了。最后再**反编译（jd-gui反编译软件）**calss文件即可看到具体的类结构了。如下如（有删减）：
```
import com.xiaocao.proxy.cglib.BookServiceBean..EnhancerByCGLIB..f2cd59c6;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;
import net.sf.cglib.core.Signature;
import net.sf.cglib.proxy.Callback;
import net.sf.cglib.proxy.MethodProxy;

public final class TempProxy extends Proxy
  implements BookServiceBean..EnhancerByCGLIB..f2cd59c6
{
  private static Method m1;
  private static Method m3;
  private static Method m7;
  private static Method m2;
  private static Method m9;
  private static Method m18;
  private static Method m20;
  private static Method m0;
  private static Method m15;
  private static Method m11;
  private static Method m14;
  private static Method m13;
  private static Method m19;
  private static Method m12;
  private static Method m4;
  private static Method m17;
  private static Method m6;
  private static Method m16;
  private static Method m8;
  private static Method m10;
  private static Method m5;

  public TempProxy(InvocationHandler paramInvocationHandler)
    throws 
  {
    super(paramInvocationHandler);
  }

  public final boolean equals(Object paramObject)
    throws 
  {
    try
    {
      return ((Boolean)this.h.invoke(this, m1, new Object[] { paramObject })).booleanValue();
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final Object newInstance(Callback[] paramArrayOfCallback)
    throws 
  {
    try
    {
      return (Object)this.h.invoke(this, m3, new Object[] { paramArrayOfCallback });
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final void query()
    throws 
  {
    try
    {
      this.h.invoke(this, m7, null);
      return;
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final void create()
    throws 
  {
    try
    {
      this.h.invoke(this, m6, null);
      return;
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

	......

  static
  {
    try
    {
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m3 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("newInstance", new Class[] { Class.forName("[Lnet.sf.cglib.proxy.Callback;") });
      m7 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("query", new Class[0]);
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      m9 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("getCallback", new Class[] { Integer.TYPE });
      m18 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("getClass", new Class[0]);
      m20 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("notifyAll", new Class[0]);
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      m15 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("wait", new Class[0]);
      m11 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("CGLIB$SET_STATIC_CALLBACKS", new Class[] { Class.forName("[Lnet.sf.cglib.proxy.Callback;") });
      m14 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("setCallbacks", new Class[] { Class.forName("[Lnet.sf.cglib.proxy.Callback;") });
      m13 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("setCallback", new Class[] { Integer.TYPE, Class.forName("net.sf.cglib.proxy.Callback") });
      m19 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("notify", new Class[0]);
      m12 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("CGLIB$SET_THREAD_CALLBACKS", new Class[] { Class.forName("[Lnet.sf.cglib.proxy.Callback;") });
      m4 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("newInstance", new Class[] { Class.forName("[Ljava.lang.Class;"), Class.forName("[Ljava.lang.Object;"), Class.forName("[Lnet.sf.cglib.proxy.Callback;") });
      m17 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("wait", new Class[] { Long.TYPE });
      m6 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("create", new Class[0]);
      m16 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("wait", new Class[] { Long.TYPE, Integer.TYPE });
      m8 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("CGLIB$findMethodProxy", new Class[] { Class.forName("net.sf.cglib.core.Signature") });
      m10 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("getCallbacks", new Class[0]);
      m5 = Class.forName("com.xiaocao.proxy.cglib.BookServiceBean$$EnhancerByCGLIB$$f2cd59c6").getMethod("newInstance", new Class[] { Class.forName("net.sf.cglib.proxy.Callback") });
      return;
    }
    catch (NoSuchMethodException localNoSuchMethodException)
    {
      throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
    }
    catch (ClassNotFoundException localClassNotFoundException)
    {
      throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
    }
  }
}
```
### 4.4 补充
>现在，如果我们需要进行半开放的权限验证，也就是**对于create需要权限验证，而对已query则不需要**。
>当然，我们完全可以在MyCglibProxy的intercept方法进行判断，如果是create方法才进行验证。这样做是可行的，但是似乎有点**违反单一职责的编程原则**，更好的方法应该是引入**filter机制**，实现**过滤验证**。下面我们将引入一个MyProxyFilter 类。
```
package com.xiaocao.proxy.cglib;

import java.lang.reflect.Method;

import net.sf.cglib.proxy.CallbackFilter;

public class MyProxyFilter implements CallbackFilter{

	@Override
	public int accept(Method method) {
		if(!"query".equalsIgnoreCase(method.getName()))     
            return 0;     
        return 1;  
	}	
}
```
>另外，我们需要将此拦截器应用到工厂方法中，让其生效。
```
Enhancer enhancer = new Enhancer();
		// 将Enhancer中的superclass属性赋值成BookServiceBean
		enhancer.setSuperclass(BookServiceBean.class);
		// 将Enhancer中的callbacks属性赋值成myProxy
		enhancer.setCallbacks(new Callback[]{myProxy,NoOp.INSTANCE});
		// 将Enhancer中的callbackfilter属性赋值成MyProxyFilter
		enhancer.setCallbackFilter(new MyProxyFilter());
		return (BookServiceBean) enhancer.create();
```
>接下来就是测试了
```
BookServiceBean service = BookServiceFactory.getProxyInstance(new MyCglibProxy("张三"));  
		service.create(); 
	
		BookServiceBean service2 = BookServiceFactory.getProxyInstance(new MyCglibProxy("李四"));  
		service2.create();
		service2.query();
```
>输出结果为：
![这里写图片描述](http://img.blog.csdn.net/20160516182207600)


>**注意:** setCallbacks中定义了所使用的拦截器，其中NoOp.INSTANCE是CGlib所提供的实际是一个没有任何操作的拦截器， 
   **他们是有序的,一定要和CallbackFilter里面的顺序一致**。上面return返回(0/1)的就是返回的顺序。也就是说如果调用query方法就使用NoOp.INSTANCE进行拦截。
## 五、动态代理的认识
>动态代理的实现可以说是一个比较综合的Java技术应用实现，他的作用是显著的，完全提供了一种AOP面向切面编程的可行性，也为远程过程调用提供了基础支持。总结一下运用的知识：
>1. **多态和继承**。代理对象和被代理对象的继承关系，实现了重写被代理对象所有方法，有选择性的运行逻辑代码。另外，工厂返回的是子类，让使用者就像调用被代理对象一样方便，这也是多态的体现。
>2. **反射机制**。动态代理里面大量运用了反射机制，比如对被代理对象的反射，对代理对象的class进行反射，对调用父类方法的反射等等。
>3. **动态生成class**。这里需要了解JVM生成calss很多细节，好在CGLIB做到了很好的封装底层的ASM实现。这也是实现动态代理的核心。
## 六、优缺点
**优点**
> 有点是显然的，这里就不在赘述了，前面提到很多次了。
**缺点**
>1. 因为动态代理的底层实现需要用到反射机制和动态生成并加载class，所以**效率会大打折扣**。
>所以，在我们实际应用中，**应该谨慎使用**。
>2. 应为需要重写被代理对象的所有方法，且需要继承被代理对象。所以**被代理对象必须不能是fianl修饰**的，**属性方法都不可以**。另外，java也有一套实现动态代理的机制，这里就不在赘述了。需要注意的是，java的动态代理与CGLIB不同，**java的实现需要被代理对象实现指定的接口**。

在此感谢[山的那边——cglib动态代理介绍（一）](http://blog.csdn.net/xiaohai0504/article/details/6832990)。

  -------------------------------------------
  >**注意**：转载请标明，转自[itboy-木小草](http://muxiaocao.cn)。
>**尊重原创，尊重技术。**