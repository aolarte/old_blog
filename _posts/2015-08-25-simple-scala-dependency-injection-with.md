---
permalink: /:categories/:year/:month/:title.html
layout: post
title: Simple Scala dependency injection with Scaldi (part 1)
date: '2015-08-25T16:47:00.000-07:00'
author: Andres Olarte
tags:
- scaldi
- dependency injection
- scala
modified_time: '2015-08-25T16:54:38.076-07:00'
blogger_id: tag:blogger.com,1999:blog-3306197464901287625.post-1474548277654738717
blogger_orig_url: http://www.javaprocess.com/2015/08/simple-scala-dependency-injection-with.html
---


<img border="0" src="http://www.scala-lang.org/resources/img/smooth-spiral.png" height="200" width="135"  align="left" hspace="10" />
Dependency Inject is a very useful pattern in most medium to larger projects. However, it's also sometimes useful in smaller projects. In smaller projects, I normally prefer to use appropriately smaller frameworks. In the case of Scala, [Scaldi](http://scaldi.org/) provides a very lightweight dependency injection mechanism.
Whenever I try to learn a new framework, I always like to learn my way up from a very simple example. The following is close to the absolute minimum example I could write to get Scaldi working. From this very simple example I will continue to add other useful features, such as testing and integration with other frameworks.

The build.sbt is very simple, we just added a single dependency:

~~~
name := "scaldi-test"

version := "1.0"

scalaVersion := "2.11.6"

libraryDependencies += "org.scaldi" %% "scaldi" % "0.5.6"
~~~

The main Scala file is shown next, and contains all of the necessary parts to get Scaldi working:

~~~scala
import scaldi.{Injector, Injectable, Module}

object HelloScaldi {
  def main(args: Array[String]) {

    val test=new Test;
    test.run
  }
}

class Test( )  extends Injectable {
  def run: Unit = {
    implicit  val injector:Injector = new UserModule //1
    val output:IService=inject[IService] //2
    println(output.execute("Scaldi"));
  }
}

class UserModule extends Module { //3
  bind [ITransport] to new MessageTransport //4
  bind [IService] to  new MessageService(inject[ITransport]) //5

}

trait ITransport {
  def send(s: String)
}

trait IService {
  def execute(x: String): String
}

class MessageService(transport:ITransport) extends IService {

  override def execute(x: String): String = {
    val ret="Hello " + x
    transport.send(ret)
    return  ret
  }
}

class MessageTransport() extends ITransport {
  override def send(s: String) = println("Sending message: " + s)
}
~~~




1. In this line, we create an Injector using a new UserModule. The injector is the entry point into the DI container. The Module defines the bindings that will be using for the injection
2. The inject method uses the implicit Injector to provide us with the object bound to trait IService.
3. The Module provides explicit bindings.
4. In this line, we bind ITransport to a new instance of MessageTransport
5. The trait IService is bound to a new instance of MessageService, in which we're injecting ITransport. This allows us to keep MessageService and MessageTransport decoupled.

This simple example can be executed using sbt:

~~~
sbt run
~~~

The output will look something like this:

~~~
[info] Running HelloScaldi
Sending message: Hello Scaldi
Hello Scaldi
[success] Total time: 2 s, completed Aug 25, 2015 7:40:23 PM
~~~

As you can see, the example is very simple, but if you want to start from the ground up, hopefully this will give you a good starting point. If you have any questions, don't hesitate to leave a comment.
You can check out the full source code [here](https://github.com/aolarte/scaldi-test).