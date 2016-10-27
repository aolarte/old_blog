---
layout: post
title: Understanding Magic Dust in Java and the JVM
date: '2016-05-31T17:22:00.000-07:00'
author: Andres Olarte
comments: true
tags:
- classloader
- bytecode
- aop
- proxy
- java
---

One of main skills of a seasoned Java developer is understanding how things work under the hood. 
There are many things that developers take for granted, without understanding how they really work.
One of the questions I get every so often is:

> Where do I put the behaviour for an annotation I just created?

The answer is to this question is fairly complicated. First, because annotations don't have any behaviour. 
They're just metadata. Second, becuase it will normally lead to something along the lines of:

> But annotations do seem to have behaviour, just look at all the stuff that happens when I mark a method @Transactional, @RolesAllowed, etc... 

And that's a very valid point, some annotations do appear to have behavior. 
But in reality, annotations are just the magic dust that we sprinkle in our code, indirectly causing all kinds of black magic to happen on the background. 

The truth is that this service class that you annotated with with `@Transactional` will only exhibit transactional behaviour in very particular cases. 
Instantiating your class using the `new` operator will NOT add transactional behaviour regardless of the presence of `@Transactional` annotations.
So how do this actually work?  Let's grab our magic dust and fly away to Neverland to find out...

## 

## Dynamic Proxies

Dynamic Proxies are a special kind of object. These object implement one or more interfaces, but their behaviour is defined not by a class but by an `InvokatinHandler`. 
Any call to any of the methods of the declared interfaces will be dispatched to the InvocationHandler. Dynamic Prox

For example, a very simple InvocationHandler would look like this:

~~~ java
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

~~~ java
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

~~~ java
proxy.getClass(): class com.sun.proxy.$Proxy0
proxy instanceof Runnable: true
Runnable.class.isAssignableFrom(proxy.getClass()): true
~~~



## Bytecode Manipulation

Dynamic Proxies were only first introduced in Java version 1.3, and even then their performance in the early days was far from ideal. 
Before Dynamic Proxies became a truly viable option, other mechanisms were devised by the Java community to apply these kinds of behaviours (even before the advent of annotations in Java 5).
The solution was to create Java classes on the fly, to provide the desired functionality. 
To achieve this, libraries were created to manipulate Java bytecode, which is the instruction set of the Java virtual machine.
The most established byte code manipulation libraries are [ASM](http://asm.ow2.org/) and [CGLIB](https://github.com/cglib/cglib). 
However newer and easier to use libraries have recently emerged, such as [Javassist](http://jboss-javassist.github.io/javassist/) 



## Compiler plugins

