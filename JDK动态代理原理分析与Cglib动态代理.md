# JDK动态代理与Cglib动态代理

## 写在前面

​	最近精神紧绷，归根到底还是自己有很多要学习的，但是又不想动。感觉要努力却努力不起来。写个笔记都是拖拖拉拉。思来想去，为了今后的辉煌生活督促自己得好好学习。从今天起，开始每周必须有2-3篇博客来巩固自己的知识，我也会从最开始一步一步打牢基础，让自己充实起来。把努力留给今天，把机遇留给明天。

## 为什么要代理？

很多时候我们想给自己的方法添加比如日志打印，性能监控。传统的做法是在每一个方法开始写，结尾写。但是这种方法会导致我们的业务代码嵌入很多无关的代码。而且重复性很高。

定义：为其他对象提供一个代理以控制对某个对象的访问，即通过代理对象访问目标对象.这样做的好处是:可以在目标对象实现的基础上,增强额外的功能操作,即扩展目标对象的功能.

代理模式的关键点是:代理对象与目标对象.代理对象是对目标对象的扩展,并会调用目标对象

## 静态代理

静态代理应用场景不大，在此不作讲解。

## 基于JDK的动态代理

### demo

1. 需要代理的接口

```java
public interface IHelloWorld {
    void sayHello();
}
```

2. 实际需要代理的类

```java
public class HelloWorldImpl implements IHelloWorld {
    public void sayHello() {
        System.out.println("hello world");
    }
}
```

3. 实现调用处理器接口

```java
public class MyInvocationHandler implements InvocationHandler {
   
    private Object target;
    public MyInvocationHandler(Object target){
        this.target = target;
    }
     /**
     * 该方法负责集中处理动态代理类上的所有方法调用。
     * 调用处理器根据这三个参数进行预处理或分派到委托类实例上反射执行
     *
     * @param proxy  代理类实例
     * @param method 被调用的方法对象
     * @param args   调用参数
     * @return
     * @throws Throwable
     */
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("代理开始");
        Object invoke = method.invoke(target, args);
        System.out.println("代理结束");
        return invoke;
    }
}
```

4. 测试类

```java
 /**
         * InvocationHandlerImpl 实现了 InvocationHandler 接口，并能实现方法调用从代理类到委托类的分派转发
         * 其内部通常包含指向委托类实例的引用，用于真正执行分派转发过来的方法调用.
         * 即：要代理哪个真实对象，就将该对象传进去，最后是通过该真实对象来调用其方法
         */
IHelloWorld helloWorld = (IHelloWorld) Proxy.newProxyInstance(Client.class.getClassLoader(), new Class<?>[]{IHelloWorld.class}, new MyInvocationHandler(new HelloWorldImpl()));
helloWorld.sayHello();
```

或者

```java
            // 1. 生成一个继承Proxy并实现IHelloWorld接口的类
            Class<?> proxyClass = Proxy.getProxyClass(Client.class.getClassLoader(), IHelloWorld.class);
			// 接下来就是利用反射来获取对象，但需要知道生成的类是需要如何创建
            // 2. 获取入参类型为InvocationHandler 的构造函数
            final Constructor<?> constructor = proxyClass.getConstructor(InvocationHandler.class);
            // 3. 实例化InvocationHandler
            final InvocationHandler ih = new MyInvocationHandler(new HelloWorldImpl());
            // 4. 传入参数获取实例对象
            IHelloWorld helloWorld = (IHelloWorld) constructor.newInstance(ih);
            helloWorld.sayHello();
```



输出结果：

![1551425578683](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1551425578683.png)

原理：在运行期间，通过反射机制来创建一个实现了一组给定接口的新类。就是生成一个class

### 动态生成的类

只需要知道如何生成的一个代理类和生成的代理类是如何的便能理解其原理

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

import cn.willvi.proxy.jdk.HelloWorldImpl;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class Client$$$$011 extends Proxy implements HelloWorldImpl {
    private static Method m1;
    private static Method m3;
    private static Method m8;
    private static Method m2;
    private static Method m6;
    private static Method m5;
    private static Method m7;
    private static Method m9;
    private static Method m0;
    private static Method m4;

    public Client$$$$011(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void sayHello() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void notify() throws  {
        try {
            super.h.invoke(this, m8, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void wait(long var1) throws InterruptedException {
        try {
            super.h.invoke(this, m6, new Object[]{var1});
        } catch (RuntimeException | InterruptedException | Error var4) {
            throw var4;
        } catch (Throwable var5) {
            throw new UndeclaredThrowableException(var5);
        }
    }

    public final void wait(long var1, int var3) throws InterruptedException {
        try {
            super.h.invoke(this, m5, new Object[]{var1, var3});
        } catch (RuntimeException | InterruptedException | Error var5) {
            throw var5;
        } catch (Throwable var6) {
            throw new UndeclaredThrowableException(var6);
        }
    }

    public final Class getClass() throws  {
        try {
            return (Class)super.h.invoke(this, m7, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void notifyAll() throws  {
        try {
            super.h.invoke(this, m9, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void wait() throws InterruptedException {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | InterruptedException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m3 = Class.forName("cn.willvi.proxy.jdk.HelloWorldImpl").getMethod("sayHello");
            m8 = Class.forName("cn.willvi.proxy.jdk.HelloWorldImpl").getMethod("notify");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m6 = Class.forName("cn.willvi.proxy.jdk.HelloWorldImpl").getMethod("wait", Long.TYPE);
            m5 = Class.forName("cn.willvi.proxy.jdk.HelloWorldImpl").getMethod("wait", Long.TYPE, Integer.TYPE);
            m7 = Class.forName("cn.willvi.proxy.jdk.HelloWorldImpl").getMethod("getClass");
            m9 = Class.forName("cn.willvi.proxy.jdk.HelloWorldImpl").getMethod("notifyAll");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
            m4 = Class.forName("cn.willvi.proxy.jdk.HelloWorldImpl").getMethod("wait");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}

```

## CGLIB动态代理

### 代码

1. 需要代理的类

```java
public class HelloCglib {
    public void sayHello(){
        System.out.println("hello cglib");
    }
}

```

2. 实现调用处理器接口实现 `MethodInterceptor`接口

```java
public class MyMethodInterceptor implements MethodInterceptor {

    public Object getProxy(Object target){
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(target.getClass());
        enhancer.setCallback(this);
        return enhancer.create();
    }

    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("代理开始");
        Object invoke = methodProxy.invokeSuper(o, objects);
        System.out.println("代理结束");
        return invoke;
    }
}
```

1. 测试类

```java
 public static void main(String[] args) {
        for (int i=0; i< 100000; i++){
            HelloCglib helloCglib = (HelloCglib) new MyMethodInterceptor().getProxy(new HelloCglib());
            helloCglib.sayHello();
        }
    }
```



## 总结

spring aop的原理其实就是动态代理，在有相同接口时使用jdk代理，没有时使用cglib代理

在jdk1.7以后 jdk代理是比cglib的效率要高的