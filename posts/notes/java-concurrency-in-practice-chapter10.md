---
layout: default
title: 第10章，避免活跃性危险
---

### 第10章，避免活跃性危险

#### 10.1.2　动态的锁顺序死锁
>在制定锁的顺序时，可以使用System.identityHashCode方法，该方法将返回由Object.hashCode返回的值。该版本中使用了System.identityHashCode来定义锁的顺序。虽然增加了一些新的代码，但却消除了发生死锁的可能性。  
```java
private static final Object tieLock = new Object();
public void transferMoney(final Account fromAcct,
		final Account toAcct,
		final DollarAmount amount)
		throws InsufficientFundsException {

	int fromHash = System.identityHashCode(fromAcct);
	int toHash = System.identityHashCode(toAcct);

	if (fromHash < toHash) {
		synchronized (fromAcct) {
			synchronized (toAcct) {
				new Helper().transfer();
			}
		}
	} else if (fromHash > toHash) {
		synchronized (toAcct) {
			synchronized (fromAcct) {
				new Helper().transfer();
			}
		}
	} else {
		synchronized (tieLock) {
			synchronized (fromAcct) {
				synchronized (toAcct) {
					new Helper().transfer();
				}
			}
		}
	}
}
```

~当无法通过键值确定加锁的顺序时，可以根据键值生成哈希码，然后根据哈希码确定加锁的顺序，这种方式也不错。在哈希码相同时，通过’加时锁‘保证只有一个线程以未知的顺序获取锁。


#### 10.1.5　资源死锁
>另一种基于资源的死锁形式就是线程饥饿死锁（Thread-StarvationDeadlock）。一个示例：一个任务提交另一个任务，并等待被提交任务在单线程的Executor中执行完成。这种情况下，第一个任务将永远等待下去，并使得另一个任务以及在这个Executor中执行的所有其他任务都停止执行。如果某些任务需要等待其他任务的结果，那么这些任务往往是产生线程饥饿死锁的主要来源，有界线程池/资源池与相互依赖的任务不能一起使用。

~记住这一点：有界线程池/资源池与相互依赖的任务不能一起使用。


#### 10.2.2　通过线程转储信息来分析死锁
>线程转储包括各个运行中的线程的栈追踪信息，这类似于发生异常时的栈追踪信息。线程转储还包含加锁信息，例如每个线程持有了哪些锁，在哪些栈帧中获得这些锁，以及被阻塞的线程正在等待获取哪一个锁。在生成线程转储之前，JVM将在等待关系图中通过搜索循环来找出死锁。如果发现了一个死锁，则获取相应的死锁信息，例如在死锁中涉及哪些锁和线程，以及这个锁的获取操作位于程序的哪些位置。  
要在UNIX平台上触发线程转储操作，可以通过向JVM的进程发送SIGQUIT信号（kill-3），或者在UNIX平台中按下Ctrl-\键，在Windows平台中按下Ctrl-Break键。在许多IDE（集成开发环境）中都可以请求线程转储。

~这个很重要，需要经常看看。
