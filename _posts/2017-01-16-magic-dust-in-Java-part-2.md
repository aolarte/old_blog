---
layout: post
title: Understanding Magic Dust in Java and the JVM (part 2)
date: '2017-01-16T07:00:00.000-06:00'
author: Andres Olarte
tags:
- classloader
- bytecode
- aop
- proxy
- java
---

# The technical nitty gritty 

In [part one](/2017/01/09/magic-dust-in-Java-part-1.html) we examined how the behavior associated to annotations  is injected into the application.
In the next few sections we'll see how the implementation of the behavior actually works.
 
 
## Dynamic Proxies

Dynamic Proxies are a special kind of object. These object implement one or more interfaces, but their behaviour is defined not by a class but by an `InvokatinHandler`. 
Any call to any of the methods of the declared interfaces will be dispatched to the InvocationHandler. Dynamic Prox

For example, a very simple InvocationHandler would look like this:

~~~java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class TestInvocationHandler implements InvocationHandler {
    private final Object target;

    public TestInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        long startTime = System.nanoTime();
        System.out.println("Starting profiling");
        try {
            return method.invoke(target, args);
        } finally {
            long endTime = System.nanoTime();
            System.out.println("Execution time: " + (endTime - startTime));
        }
    }
}
~~~

The code to actually create the proxy looks like this:

~~~java
import java.lang.reflect.Proxy;

public class DynamicProxyTest implements Runnable{

    public static void main(String... args) {
    
        DynamicProxyTest object = new DynamicProxyTest();
        Runnable proxy = (Runnable)Proxy.newProxyInstance(
            DynamicProxyTest.class.getClassLoader(),
            new Class[]{Runnable.class},
            new TestInvocationHandler(object));
        proxy.run();
    }
    
    public void run() {
        System.out.println("Test");
    }
}
~~~

Keeping in mind that Dynamic Proxies are special mechanisms buried deep in the JVM, here are some useful properties to keep in mind:

~~~java
proxy.getClass(): class com.sun.proxy.$Proxy0
proxy instanceof Runnable: true
Runnable.class.isAssignableFrom(proxy.getClass()): true
~~~

### Limitations

* JVM Dynamic Proxies can only proxy interfaces, so no enhancing of concrete or abstract classes
* Any and all invocations must be handled by the method interceptor
* Heavy use of reflection
* All usual limits of JVM are enforced, such as the limitation on the number of interfaces that can be implemented

## Bytecode Manipulation

Dynamic Proxies were only first introduced in Java version 1.3, and even then their performance in the early days was far from ideal. 
Before Dynamic Proxies became a truly viable option, other mechanisms were devised by the Java community to apply these kinds of behaviours (even before the advent of annotations in Java 5).
The solution was to create Java classes on the fly, to provide the desired functionality. 
To achieve this, libraries were created to manipulate Java bytecode, which is the instruction set of the Java virtual machine.
The most established byte code manipulation libraries are [ASM](http://asm.ow2.org/) and [CGLIB](https://github.com/cglib/cglib). 
However newer and easier to use libraries have recently emerged, such as [Javassist](http://jboss-javassist.github.io/javassist/) 

Bytecode manipulation ranges from very low level using [JVM opcodes](https://en.wikipedia.org/wiki/Java_bytecode_instruction_listings) to more higher level mechanisms such as CGLIB's Enhancer. 
The Enhancer will generate a dynamic subclass, which enable dynamic interception. If you ever see a class that ends in `$$EnhancerByCGLIB` in your debugger, you'll know that you're working with a class that was "Enhanced" by CGLIB.

The following example enhances a class, by writing the output in HTML paragraph tags: `<p>` and `</p>`. 

~~~java
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.FixedValue;
import net.sf.cglib.proxy.InvocationHandler;
import net.sf.cglib.proxy.MethodInterceptor;

import java.util.function.Supplier;


public class EnhancerTest implements Supplier<String> {

    public static void main(String... args) {
        System.out.println(new EnhancerTest().get());
        System.out.println(testInvocationInterceptor().get());
    }  

    public static Supplier<String> testInvocationInterceptor() {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(EnhancerTest.class);
        enhancer.setCallback((MethodInterceptor) (obj,  method,  args, proxy) -> {
            if("get".equalsIgnoreCase(method.getName())
                    && method.getReturnType() == String.class) {
                return "<p>" + proxy.invokeSuper(obj, args) + "</p>";
            } else {
                return proxy.invokeSuper(obj, args);
            }
        });
        return (Supplier) enhancer.create();

    }

    @Override
    public String get() {
        return "Hello Test!";
    }
}
~~~

The output will look like this:

~~~
Hello Test!
<p>Hello Test!</p>
~~~

## ThreadLocal, the missing link 

So how does a framework like Spring implement functionality such as transaction propagation and security? In many case the secret behind the proxies is [ThreadLocal](https://docs.oracle.com/javase/7/docs/api/java/lang/ThreadLocal.html)!

In transaction management for example, before entering a method marked as [@Transactional](https://javaee-spec.java.net/nonav/javadocs/javax/transaction/Transactional.html), the proxy code will check in a `ThreadLocal` variable to see if there's a transaction in progress. If there is, it will be reused (reentrant transactions), otherwise a new one will be created and stored in `ThreadLocal` for access down the stack.

Security annotations work in a similar fashion, with the authentication code storing the authorization information in `ThreadLocal`. This information is it's accessible by the proxy code wrapping methods annotated with (@RolesAllowed)[http://docs.oracle.com/javaee/6/api/javax/annotation/security/RolesAllowed.html]. 

While this is not a discussion about `ThreadLocal` it's worth mentioning that as the name implies the information stored in such a variable is only available to code executing in the same thread. Special logic must take care of propagating this information to other threads in the JVM, or to a remote thread in case of RMI.

## Final notes

Despite their bad rep in the early days, JDK dynamic proxies have performance that is within the realm of bytecode generation.
Nowadays the decision between using JDK Dynamic Proxies or one of the code generation libraries is not clear cut. 
For example Spring will fallback to JDK Dynamic Proxies if no code generation library is available (and as long a you're only injecting interfaces).  
Regardless of the chosen method, just keep in mind that behind the covers, most moder web application are using plenty of proxies for a variety of critical functionality!

Thanks for reading, and if you have any questions or comments, feel free to drop a note in the comment section.