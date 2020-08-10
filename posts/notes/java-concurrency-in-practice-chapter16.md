---
layout: default
title: 第16章，Java内存模型
---

### 第16章，Java内存模型

#### 16.1.3　Java内存模型简介
>Happens-Before的规则包括：  
程序顺序规则。如果程序中操作A在操作B之前，那么在线程中A操作将在B操作之前执行。  
监视器锁规则。在监视器锁上的解锁操作必须在同一个监视器锁上的加锁操作之前执行。  
volatile变量规则。对volatile变量的写入操作必须在对该变量的读操作之前之前执行。  
线程启动规则。在线程上对Thread.Start的调用必须在该线程中执行任何操作之前执行。  
线程结束规则。线程中的任何操作都必须在其他线程检测到该线程已经结束之前执行，或者从Thread.join中成功返回，或者在调用Thread.isAlive时返回false。  
中断规则。当一个线程在另一个线程上调用interrupt时，必须在被中断线程检测到interrupt调用之前执行（通过抛出InterruptedException，或者调用isInterrupted和interrupted）。  
终结器规则。对象的构造函数必须在启动该对象的终结器之前执行完成。  
传递性。如果操作A在操作B之前执行，并且操作B在操作C之前执行，那么操作A必须在操作C之前执行。  

~这个有必要经常看看，至于有啥子用处了，我也不知道，可能会在面试中遇到吧。


#### 16.2.3　安全初始化模式
>在初始器中采用了特殊的方式来处理静态域（或者在静态初始化代码块中初始化的值），并提供了额外的线程安全性保证。静态初始化器是由JVM在类的初始化阶段执行，即在类被加载后并且被线程使用之前。由于JVM将在初始化期间获得一个锁，并且每个线程都至少获取一次这个锁以确保这个类已经加载，因此在静态初始化期间，内存写入操作将自动对所有线程可见。因此无论是在被构造期间还是被引用时，静态初始化的对象都不需要显式的同步。然而，这个规则仅适用于在构造时的状态，如果对象是可变的，那么在读线程和写线程之间仍然需要通过同步来确保随后的修改操作是可见的，以及避免数据破坏。  
```java
@ThreadSafe
public class EagerInitialization {
	private static Resource resource = new Resource();
	public static Resource getResource() { return resource; }
}
//
@ThreadSafe
public class ResourceFactory {
	private static class ResourceHolder {
		public static Resource resource = new Resource();
	}
	public static Resource getResource() {
		return ResourceHolder.resource;
	}
}
```

~静态变量能够实现安全发布的原因，以及延长初始化占位符模式，也就是 ResourceHolder，这个是第一次看到。



#### 16.3　初始化过程中的安全性
>对于含有final域的对象，初始化安全性可以防止对对象的初始引用被重排序到构造过程之前。当构造函数完成时，构造函数对final域的所有写入操作，以及对通过这些域可以到达的任何变量的写入操作，都将被“冻结”，并且任何获得该对象引用的线程都至少能确保看到被冻结的值。对于通过final域可到达的初始变量的写入操作，将不会与构造过程后的操作一起被重排序。  
初始化安全性只能保证通过final域可达的值从构造过程完成时开始的可见性。对于通过非final域可达的值，或者在构成过程完成后可能改变的值，必须采用同步来确保可见性。

~下次再来看的时候，是否有新的见解。
