---
title: "动态代理 Dynamic Proxy"
date: 2020-09-02T21:44:20+08:00
draft: false
---

## 静态代理

### 原始接口和实现类

```java
package vip.housir.proxy;

/**
 * @author housirvip
 */
public interface SmsService {

    /**
     * send msg
     *
     * @param msg String
     */
    public void send(String msg);
}
```

```java
package vip.housir.proxy;

/**
 * @author housirvip
 */
public class SmsServiceImpl implements SmsService {

    @Override
    public void send(String msg) {
        System.out.println("send msg: " + msg);
    }
}
```

### 静态代理类

```java
package vip.housir.proxy;

/**
 * @author housirvip
 */
public class SmsProxy {

    private final SmsService smsService;

    public SmsProxy(SmsService sms) {
        smsService = sms;
    }

    public void send(String msg) {
        System.out.println("Static Proxy before: send");
        smsService.send(msg);
        System.out.println("Static Proxy after: send");
    }
}
```

### 手动调用

```java
import org.junit.Test;
import vip.housir.proxy.SmsProxy;
import vip.housir.proxy.SmsServiceImpl;

public class TestDynamicProxy {

    @Test
    public void testStaticProxy() {
        SmsProxy sms = new SmsProxy(new SmsServiceImpl());
        sms.send("yes sir!");
    }
}
```

### 结果输出

```shell
Static Proxy before: send
send msg: yes sir!
Static Proxy after: send

Process finished with exit code 0
```

## JDK 代理

原始类和原始接口同静态代理

### JDK 动态代理类

```java
package vip.housir.proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

/**
 * @author housirvip
 */
public class LogInvocationHandler implements InvocationHandler {

    private final Object obj;

    public LogInvocationHandler(Object obj) {
        this.obj = obj;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Dynamic Proxy before: " + method.getName());
      	// 反射调用方法， 动态增强
        Object result = method.invoke(obj, args);
        System.out.println("Dynamic Proxy after: " + method.getName());
        return result;
    }
}
```

### 代理工厂

```java
package vip.housir.proxy;

import java.lang.reflect.Proxy;

/**
 * @author housirvip
 */
public class JdkProxyFactory {

    public static Object createLogProxy(Object obj) {
        return Proxy.newProxyInstance(
                obj.getClass().getClassLoader(),
                obj.getClass().getInterfaces(),
                new LogInvocationHandler(obj)
        );
    }
}
```

### 手动调用

```java
import org.junit.Test;
import vip.housir.proxy.JdkProxyFactory;
import vip.housir.proxy.SmsService;
import vip.housir.proxy.SmsServiceImpl;

public class TestJdkDynamicProxy {

    @Test
    public void testJdkDynamicProxy() {
        SmsService sms = (SmsService) JdkProxyFactory.createLogProxy(new SmsServiceImpl());
        sms.send("yes sir!");
    }
}
```

### 结果输出

```shell
Dynamic Proxy before: send
send msg: yes sir!
Dynamic Proxy after: send

Process finished with exit code 0
```

## Cglib 代理

### POM 引入依赖

```xml
<!-- https://mvnrepository.com/artifact/cglib/cglib -->
        <dependency>
            <groupId>cglib</groupId>
            <artifactId>cglib</artifactId>
            <version>3.2.5</version>
        </dependency>
```

### 原始类

```java
package vip.housir.proxy;

/**
 * @author housirvip
 */
public class SmsSender {

    public void send(String msg){
        System.out.println("send msg: " + msg);
    }
}
```

### Cglib 代理增强类

```java
package vip.housir.proxy;

import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

/**
 * @author housirvip
 */
public class LogMethodInterceptor implements MethodInterceptor {

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("Dynamic Proxy before: " + method.getName());
        // 代理调用父类方法， 动态增强
        Object result = methodProxy.invokeSuper(o, objects);
        System.out.println("Dynamic Proxy after: " + method.getName());
        return result;
    }
}
```

### 代理工厂

```java
package vip.housir.proxy;

import net.sf.cglib.proxy.Enhancer;

/**
 * @author housirvip
 */
public class CglibProxyFactory {

    public static Object createLogProxy(Class<?> c) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(c);
        enhancer.setClassLoader(c.getClassLoader());
        //设置增强类的实现逻辑
        enhancer.setCallback(new LogMethodInterceptor());
        return enhancer.create();
    }
}
```

### 手动调用

```java
import org.junit.Test;
import vip.housir.proxy.SmsSender;
import vip.housir.proxy.CglibProxyFactory;

public class TestDynamicProxy {

    @Test
    public void testCglibDynamicProxy() {
        SmsSender sms = (SmsSender) CglibProxyFactory.createLogProxy(SmsSender.class);
        sms.send("yes sir!");
    }
}
```

### 结果输出

```shell
Dynamic Proxy before: send
send msg: yes sir!
Dynamic Proxy after: send

Process finished with exit code 0
```

## 对比

### 静态代理和动态代理

1. 动态代理更加灵活，不需要必须实现接口，可以直接代理实现类，并且可以不需要针对每个目标类都创建一个代理类。
2. 静态代理中，接口一旦新增加方法，目标对象和代理对象都要进行修改，非常麻烦。
3. 静态代理在编译时就将接口、实现类、代理类这些都变成了一个个实际的 class 文件，而动态代理是在运行时动态生成类字节码，并加载到 JVM 中。

### JDK 代理和 CGLIB 代理

1. JDK 动态代理只能只能代理实现了接口的类，而 CGLIB 可以代理未实现任何接口的类。
2. CGLIB 动态代理是通过生成一个被代理类的子类来拦截被代理类的方法调用，因此不能代理声明为 final 类型的类和方法。
3. 就二者的效率来说，大部分情况都是 JDK 动态代理更优秀，随着 JDK 版本的升级，这个优势更加明显。