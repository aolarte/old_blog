---
layout: post
title: Understanding Magic Dust in Java and the JVM (part 1)
date: '2017-01-09T07:00:00.000-06:00'
author: Andres Olarte
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

> Where do I put the behavior for an annotation I just created?

The answer to this question is fairly complicated. First, because annotations don't have any behavior. 
They're just metadata. Second, because most answers will normally lead to something along the lines of:

> But annotations do seem to have behavior, just look at all the stuff that happens when I mark a method @Transactional, @RolesAllowed, etc... 

And that's a very valid point, some annotations do appear to have behavior. I recently even heard people half jokingly reference to this as "Annotation Driven Development". But in reality, annotations are just the magic dust that we sprinkle in our code, indirectly causing all kinds of black magic to happen on the background. 

The truth is that this service class that you annotated with with `@Transactional` will only exhibit transactional behavior in very particular cases. 
Instantiating your class using the `new` operator will NOT add transactional behavior regardless of the presence of `@Transactional` annotations.
So how does it actually work?  Let's grab our magic dust and fly away to Neverland to find out...

## The basics, cross-cutting concerns and pointcuts

 Aspect-oriented programming or AOP is a programming paradigm that allows separation of cross cutting concerns from core concerns. 
 Cross cutting concerns are applied based on point cuts.
 
* **Core concerns** are the main tasks that a program is written to satisfy. Core concerns normally include the business logic.
* **Cross cutting concerns** are aspects of a system that affect the behavior of other core concerns. These include things like security, logging, transactions, etc.
* **Pointcuts** define where exactly to apply advice (inject the cross cutting concern).

While AOP never lived up to the hype it generated at some point in the past, some of its patterns have 
been widely used by frameworks to apply framework provided functionality to existing code without the need of explicit invocation.

# How does it actually work?

The behavior is really injected through means separate from the annotations, the annotations really just mark where should be done. 
We'll now look at how it's done at the Dependency Injection (DI) level, through Java Agents at runtime, and during the compile phase by using compiler annotation processors.

## Dependency Injection Container

Your favorite dependency injection framework is normally one of the ways in which annotations are associated with actual behavior. Basically the DI container will wrap the actual bean before injecting it. 

We'll see examples in the three (arguably) most popular DI frameworks. The examples will turn strings into HTML (basically just wrap the string in `<p></p>` tags).

A sample service that will be instrumented looks like this:

~~~java
@HTMLPrettify
public class TestService {
    public String buildMessage() {
        return "Hello World!";
    }
}
~~~

Calling `testService.buildMessage()` will return `<p>Hello World!</p>` if the interceptor is set up correctly.

In this case the core concern is building the message `Hello World`, while the cross cutting concern is applying some format to it.  We can apply the same cross cutting concern to multiple core concerns without changing either. We just have to define the right pointcuts.

### Spring Dependency Injection


In newer versions of Spring, adding an aspect is relatively easy:

~~~java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

@Component
@Aspect
public class HTMLPrettifyAdvice {

    @Around("@annotation(html)")
    public Object prettify(ProceedingJoinPoint pjp, HTML html) throws Throwable {
        return "<p>" + pjp.proceed() + "</p>";
    }
}
~~~

* `@Component` The class is marked as component to be added to the Spring container
* `@Aspect` The class must be marked as an aspect, using the standard Aspect4J annotations
* `@Around` The aspect is marked as an `Around` concern, and the pointcut is defined as classes annotated with `@HTML`

For the Aspect to work, Spring must be configure to enable AspectJ proxies:

~~~java
@Configuration
@ComponentScan("com.andresolarte.harness.spring4")
@EnableAspectJAutoProxy
public class Spring4TestConfig {

}
~~~

### JavaEE and CDI

CDI (Context and Dependency Injection) frameworks use the concept of `Interceptor`. 
These interceptors can do a lot of things, including providing aspect advice. 
A simple interceptor will look like this:

~~~java
import javax.interceptor.AroundInvoke;
import javax.interceptor.Interceptor;
import javax.interceptor.InvocationContext;


@Interceptor
@HTMLPrettify
public class HTMLPrettifyInterceptor {

    @AroundInvoke
    public Object prettifyOutput(InvocationContext invocationContext)
            throws Exception {
        return "<p>" + invocationContext.proceed() + "</p>";       
    }
}
~~~

* `@Interceptor` We mark the class as an interceptor, this will let CDI know the special purpose of this class.
* `@HTMLPrettify` This is the actual annotation that we want to provide behavior for. 
* `@AroundInvoke` Indicates the type of advice. This allows us access change the paramters used to invoke the underlying logic, change the return value, or return something completely different.

The annotation that we're going to bind to has a few special requirements:

~~~java
import javax.interceptor.InterceptorBinding;
import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Inherited
@InterceptorBinding
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE,ElementType.METHOD})
public @interface HTMLPrettify {
}
~~~

* `@Inherited` Meta-annotation that indicates that an annotation type is automatically inherited to subclasses.
* `@InterceptorBinding` Specifies that the annotated type should be associated with an interceptor
* `@Retention` Should normally be `RUNTIME` to ensure that the annotation is available to the JVM time. The default is `CLASS`
 

To enable the interceptor, it must be defined in the `beans.xml` file. This is needed to be able to define the order in which the interceptors will fire.

~~~xml
<beans xmlns="http://xmlns.jcp.org/xml/ns/javaee"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
        http://xmlns.jcp.org/xml/ns/javaee/beans_1_1.xsd"
       bean-discovery-mode="all">
    <interceptors>
        <class>com.andresolarte.harness.cdi.interceptor.HTMLPrettifyInterceptor</class>
    </interceptors>
</beans>
~~~


#### Adding annotations at runtime

Sometime the classes that we want to intercept don't have our special annotation. 
In those cases, an extension provides a way to add annotation to class definitions at runtime.

Let's say for example that we want to instrument third party classes that are annotated with the third party annotation `@HTML`.
We lack the means to modify the annotation or the classes. Extensions come in to help us.

An extension can observe container lifecycle events and react to them. 
For example we can observe when the CDI context discovers a class or interface.

~~~java
public class HTMLAnnotationProcessor implements Extension {

    <T> void processAnnotatedType(@Observes @WithAnnotations({HTML.class}) ProcessAnnotatedType<T> pat) {
        AnnotatedTypeBuilder annotatedTypeBuilder = new AnnotatedTypeBuilder()
                .readFromType(pat.getAnnotatedType())
                .addToClass(new AnnotationLiteral<HTMLPrettify>() {
                });
        AnnotatedType<T> type= annotatedTypeBuilder.create();
        pat.setAnnotatedType(type);
    }
}
~~~

We're also leveraging the `AnnotatedTypeBuilder` which is part of the [Apache DeltaSpike project](https://deltaspike.apache.org/).
This facilitates adding our `@HTMLPrettify` annotation to the recently discovered type.

CDI extensions use the standard Java extension mechanism, and need to be declared in file named `javax.enterprise.inject.spi.Extension` in the `META-INF/services` directory.
This file will simply list the FQN of the extensions we want to use:

~~~
com.andresolarte.harness.cdi.processor.HTMLAnnotationProcessor
~~~

After doing this, the any `@HTML` annotated objects created by the CDI container will behave as if they had `@HTMLPrettify` annotation.

### Google Guice

Google Guice uses the standard `MethodInterceptor` interface defined in the core AOP library:

~~~java
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;

public class HTMLPrettifyInterceptor implements MethodInterceptor {

    public Object invoke(MethodInvocation invocation) throws Throwable {
        Object result = invocation.proceed();
        if (invocation.getMethod().getReturnType() == String.class) {
            System.out.println("Prettifying Output call to method: "
                    + invocation.getMethod().getName());
            result = "<p>" + result + "</p>";
        }
        return result;
    }
}
~~~

Interceptors (the cross-cutting concern) will need to be registered in one of your modules, including a `Matcher` specifying which objects to intercept (the point-cut definition).

~~~java
public class TestModule extends AbstractModule {
    @Override
    protected void configure() {
        bindInterceptor(Matchers.any(), Matchers.annotatedWith(HTMLPrettify.class),
                new HTMLPrettifyInterceptor());
    }
}
~~~

In this case we're binding the interceptor to any class that has a method annotated with `@HTMLPrettify`.

## Java Agents

Java agents can be configured at startup, and provide mechanisms to change classes as they're being loaded by the JVM.
These agents exist on a layer that is closer to the bare metal than the class loader.

~~~java
public class TestAgent {

    public static void premain(String agentArguments,
                               Instrumentation instrumentation){
        ClassFileTransformer transformer=new TestTransformer();
        instrumentation.addTransformer(transformer);
    }

    public static class TestTransformer  implements ClassFileTransformer {

        public byte[] transform(ClassLoader loader, 
                                String className, 
                                Class<?> classBeingRedefined, 
                                ProtectionDomain protectionDomain, 
                                byte[] classfileBuffer) throws IllegalClassFormatException {
            byte[] ret=classfileBuffer;
            if (shouldTransform(className)) {
                ret=transform(classfileBuffer);
            }
            return ret;
        }
    }
}
~~~

Java agents provide access to the raw bytes that make up the class. The same bytes you would see in a `.class` file. 
While you're welcome to modify the byte array by hand, it's more practical to use a byte code manipulation library. 
A complete example using Javassist can be seen [here](https://github.com/aolarte/minitrue).

Java agents must be specified in the command line when starting the JVM process:

    java -javaagent:myagent.jar -jar myapp.jar

### Limitations    

* Some classes [cannot be modified](http://docs.oracle.com/javase/8/docs/api/java/lang/instrument/Instrumentation.html#isModifiableClass-java.lang.Class-). For example, primitive classes (for example, java.lang.Integer.TYPE) and array classes are never modifiable.
* By the time the agent runs, some classes have already been loaded. The Instrumentation API does provide a way to redefine classes, but it's an [optional feature of the JVM](http://docs.oracle.com/javase/8/docs/api/java/lang/instrument/Instrumentation.html#isRedefineClassesSupported--).

## Annotation processors

Annotation processors are executed by the standard java compiler `javac`. These processors allow extensions to apply validation rules or generate new resources. Tools such as Dagger and Lombok make their magic happen using annotation processors.

Compile time annotation processors are registered using the standard Java extension mechanism. These extensions are declared in file named `javax.annotation.processing.Processor` in the `META-INF/services` directory. This file will simply list the FQN of the extensions we want to use:

~~~
com.andresolarte.compile.processor.HTMLAnnotationProcessor
~~~

Any jar file in the compiler classpath will be scanned by the compiler to determine if it declares one or more annotation processors.

The actual annotation processor must extend [AbstractProcessor](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/AbstractProcessor.html):

~~~java
@SupportedSourceVersion(SourceVersion.latestSupported())
@SupportedAnnotationTypes({
   "com.andresolarte.compile.processor.HTMLPrettify"
 })
public class HTMLAnnotationProcessor extends AbstractProcessor {

	@Override
	public synchronized void init(ProcessingEnvironment env){ }

	@Override
	public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) { }

}
~~~

* `@SupportedSourceVersion` Indicates the Java source version that the processor is compatible with.
* `@SupportedAnnotationTypes` Specifies that the listed annotation types should be processed.
* `init` A method called once per processor, allowing for any initialization.
* `process` The method where the annotations are actually processed. This method can create new resources, or signal an error to the `javac` process that will halt the compilation process.
 

### Lombok 
Lombok has become one of the most popular compiler plugins, given all of the functionality that it provides with just a few annotations.

Of special note is that compiler plugins are not supposed to change the AST (Abstract Syntax Tree). Compiler plugins can validate the resources, they can create new resources, but (in theory) they can't change the existing code.

So how does Lombok work? It uses undocumented APIs in the HotSpot compiler and Eclipse's JDT compiler to modify the AST.  

While the functionality provided by the library is very valuable, it's worth keeping in mind that it depends on undocumented functionality that might prevent upgrading to newer or different compilers. More discussion of these controversies can be read [here](http://jnb.ociweb.com/jnb/jnbJan2010.html#controversy).

## to be continued...

In [part two](/2017/01/16/magic-dust-in-Java-part-2.html) see how the behavior is actually implemented. We'll look at Dynamic Proxies and Bytecode Manipulation.

Feel free to drop a note in the comments section.
