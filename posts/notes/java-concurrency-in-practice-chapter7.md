---
layout: default
title: 第7章，取消与关闭
---

### 第7章，取消与关闭

#### 7.1.1　中断?
>每个线程都有一个boolean类型的中断状态。当中断线程时，这个线程的中断状态将被设置为true。在Thread中包含了中断线程以及查询线程中断状态的方法。interrupt方法能中断目标线程，而isInterrupted方法能返回目标线程的中断状态。静态的interrupted方法将清除当前线程的中断状态，并返回它之前的值，这也是清除中断状态的唯一方法。  
阻塞库方法，例如Thread.sleep和Object.wait等，都会检查线程何时中断，并且在发现中断时提前返回。它们在响应中断时执行的操作包括：清除中断状态，抛出InterruptedException，表示阻塞操作由于中断而提前结束。JVM并不能保证阻塞方法检测到中断的速度，但在实际情况中响应速度还是非常快的。  
当线程在非阻塞状态下中断时，它的中断状态将被设置，然后根据将被取消的操作来检查中断状态以判断发生了中断。通过这样的方法，中断操作将变得“有黏性”——如果不触发InterruptedException，那么中断状态将一直保持，直到明确地清除中断状态。  

~自己处理中断时，如果不抛出InterruptedException异常，或者显式清除线程的中断状态，那么线程的中断状态将一直保持。是不是这样，自己还真是需要测试一下。


>在使用静态的interrupted时应该小心，因为它会清除当前线程的中断状态。如果在调用interrupted时返回了true，那么除非你想屏蔽这个中断，否则必须对它进行处理—可以抛出InterruptedException，或者通过再次调用interrupt来恢复中断状态，

~记录一下，用于提醒。


#### 7.1.2　中断策略#
>最合理的中断策略是某种形式的线程级（Thread-Level）取消操作或服务级（Service-Level）取消操作：尽快退出，在必要时进行清理，通知某个所有者该线程已经退出。此外还可以建立其他的中断策略，例如暂停服务或重新开始服务，

~什么最合理的中断策略，值得看看，以后就可以按此去做。

>这就是为什么大多数可阻塞的库函数都只是抛出InterruptedException作为中断响应。它们永远不会在某个由自己拥有的线程中运行，因此它们为任务或库代码实现了最合理的取消策略：尽快退出执行流程，并把中断信息传递给调用者，从而使调用栈中的上层代码可以采取进一步的操作。

~为什么要抛出InterruptedException作为中断响应呢？这里给出了理由。



#### 7.1.3　响应中断
>如果不想或无法传递InterruptedException（或许通过Runnable来定义任务），那么需要寻找另一种方式来保存中断请求。一种标准的方法就是通过再次调用interrupt来恢复中断状态。

~另外一种响应中断的方式，应该知道。


#### 7.1.6　处理不可中断的阻塞#
>然而，并非所有的可阻塞方法或者阻塞机制都能响应中断；如果一个线程由于执行同步的SocketI/O或者等待获得内置锁而阻塞，那么中断请求只能设置线程的中断状态，除此之外没有其他任何作用。对于那些由于执行不可中断操作而被阻塞的线程，可以使用类似于中断的手段来停止这些线程，但这要求我们必须知道线程阻塞的原因。  
Java.io包中的同步SocketI/O。在服务器应用程序中，最常见的阻塞I/O形式就是对套接字进行读取和写入。虽然InputStream和OutputStream中的read和write等方法都不会响应中断，但通过关闭底层的套接字，可以使得由于执行read或write等方法而被阻塞的线程抛出一个SocketException。  
Java.io包中的同步I/O。当中断一个正在InterruptibleChannel上等待的线程时，将抛出ClosedByInterruptException并关闭链路（这还会使得其他在这条链路上阻塞的线程同样抛出ClosedByInterruptException）。当关闭一个InterruptibleChannel时，将导致所有在链路操作上阻塞的线程都抛出AsynchronousCloseException。大多数标准的Channel都实现了InterruptibleChannel。  
Selector的异步I/O。如果一个线程在调用Selector.select方法（在java.nio.channels中）时阻塞了，那么调用close或wakeup方法会使线程抛出ClosedSelectorException并提前返回。  
获取某个锁。如果一个线程由于等待某个内置锁而阻塞，那么将无法响应中断，因为线程认为它肯定会获得锁，所以将不会理会中断请求。但是，在Lock类中提供了lockInterruptibly方法，该方法允许在等待一个锁的同时仍能响应中断。

~提供的方法无非就是关闭套接字，关闭可中断通道上阻塞的线程或直接关闭通道（可中断通道），或者在selector上调用close或wakeup方法，或者使用Lock代替内置锁来处理不可中断的阻塞。



#### 7.1.7　采用newTaskFor来封装非标准的取消？
>我们可以通过newTaskFor方法来进一步优化ReaderThread中封装非标准取消的技术，这是Java 6在ThreadPoolExecutor中的新增功能。当把一个Callable提交给ExecutorService时，submit方法会返回一个Future，我们可以通过这个Future来取消任务。newTaskFor是一个工厂方法，它将创建Future来代表任务。newTaskFor还能返回一个RunnableFuture接口，该接口扩展了Future和Runnable（并由FutureTask实现）。  
通过定制表示任务的Future可以改变Future.cancel的行为。例如，定制的取消代码可以实现日志记录或者收集取消操作的统计信息，以及取消一些不响应中断的操作。通过改写interrupt方法，ReaderThread可以取消基于套接字的线程。同样，通过改写任务的Future.cancel方法也可以实现类似的功能。  
​```java
//定义自己的任务类接口
public interface CancellableTask<T> extends Callable<T> {
	void cancel();
	RunnableFuture<T> newTask();
}

@ThreadSafe
public class CancellingExecutor extends ThreadPoolExecutor {
	...
	protected<T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
		if (callable instanceof CancellableTask)
			return ((CancellableTask<T>) callable).newTask();
		else
			return super.newTaskFor(callable);
	}
}

//创建自己的任务类，并且在cancel方法中关闭socket，达到中断任务的作用。
public abstract class SocketUsingTask<T>
		implements CancellableTask<T> {

	@GuardedBy("this") private Socket socket;

	protected synchronized void setSocket(Socket s) { socket = s; }

	public synchronized void cancel() {
		try {
			if (socket != null)
				socket.close();
		} catch (IOException ignored) { }
	}

	public RunnableFuture<T> newTask() {
		return new FutureTask<T>(this) {
			public boolean cancel(boolean mayInterruptIfRunning) {
				try {
					SocketUsingTask.this.cancel();
				} finally {
					return super.cancel(mayInterruptIfRunning);
				}
			}
		};
	}
}
 ```

~其实就是通过改写 ThreadPoolExecutor中的 newTaskFor方法来定制FutureTask的行为，这样就可以覆盖FutureTask中cacel方法，对于不能响应中断的阻塞操作，可以通过特殊的方式取消任务。



#### 7.2.3　“毒丸”对象
>只有在生产者和消费者的数量都已知的情况下，才可以使用“毒丸”对象。在Indexing-Service中采用的解决方案可以扩展到多个生产者：只需每个生产者都向队列中放入一个“毒丸”对象，并且消费者仅当在接收到N producers个“毒丸”对象时才停止。这种方法也可以扩展到多个消费者的情况，只需生产者将N consumers个“毒丸”对象放入队列。然而，当生产者和消费者的数量较大时，这种方法将变得难以使用。只有在无界队列中，“毒丸”对象才能可靠地工作。

~扩展到多个消费者的情况，只需将消费者数量的毒丸放入队列即可（有N个消费者，生产者就放入N个毒丸），这样生产者放入毒丸后，就可以退出了，另外每个消费者在获取到毒丸后，也都可以全部退出。



#### 7.2.5　shutdownNow的局限性
>当通过shutdownNow来强行关闭ExecutorService时，它会尝试取消正在执行的任务，并返回所有已提交但尚未开始的任务，从而将这些任务写入日志或者保存起来以便之后进行处理。  
然而，我们无法通过常规方法来找出哪些任务已经开始但尚未结束。这意味着我们无法在关闭过程中知道正在执行的任务的状态，除非任务本身会执行某种检查。要知道哪些任务还没有完成，你不仅需要知道哪些任务还没有开始，而且还需要知道当Executor关闭时哪些任务正在执行。  
​```java
public class TrackingExecutor extends AbstractExecutorService {
	private final ExecutorService exec;
	private final Set<Runnable> tasksCancelledAtShutdown =
			Collections.synchronizedSet(new HashSet<Runnable>());

	...

	//调用者线程通过此方法可以获取已开始运行且中断的任务。
	public List<Runnable> getCancelledTasks() {
		if (!exec.isTerminated())
			throw new IllegalStateException(...);
		return new ArrayList<Runnable>(tasksCancelledAtShutdown);
	}

	public void execute(final Runnable runnable) {
		exec.execute(new Runnable() {
		public void run() {
			try {
				runnable.run();
			} finally {
			//如果线程池已关闭，且执行此任务的线程已中断，则放入任务集合中。
				if (isShutdown()
					&& Thread.currentThread().isInterrupted())
				tasksCancelledAtShutdown.add(runnable);
			}
		}
		});
	}
	// delegate other ExecutorService methods to exec
}


public abstract class WebCrawler {
	private volatile TrackingExecutor exec;

	public synchronized void stop() throws InterruptedException {
		try {
			saveUncrawled(exec.shutdownNow());
		if (exec.awaitTermination(TIMEOUT, UNIT))
			saveUncrawled(exec.getCancelledTasks());
		} finally {
			exec = null;
		}
	}
}
 ```

~shutdownNow关闭ExecutorService时，才关闭操作会返回队列中尚未运行的任务的列表，但是不会返回已开始运行且未完成的任务。上例就通过try，finally将任务的执行包裹起来，并在finally块中，通过检查线程池的状态以及当前任务的执行线程的中断状态来确定任务执行是否中断，如果检查为真，则将任务放入列表。调用者线程通过获取这个列表就可以知道哪些任务中断了。


#### 7.3　处理非正常的线程终止#
>像Swing事件线程这样的服务可能只是因为某个编写不当的事件处理器抛出NullPointerException而失败，这种情况是非常糟糕的。因此，这些线程应该在try-catch代码块中调用这些任务，这样就能捕获那些未检查的异常了，或者也可以使用try-finally代码块来确保框架能够知道线程非正常退出的情况，并做出正确的响应。在这种情况下，你或许会考虑捕获RuntimeException，即当通过Runnable这样的抽象机制来调用未知的和不可信的代码时。  
在Thread API中同样提供了UncaughtExceptionHandler，它能检测出某个线程由于未捕获的异常而终结的情况。这两种方法是互补的，通过将二者结合在一起，就能有效地防止线程泄漏问题。  

~处理非正常的线程终止，这里提供了三类方法，一是在try-catch代码块中调用这些任务，二是在try-finally代码块能够知道线程何时已经退出，可以在finally块中做相应的处理，三是使用UncaughtExceptionHandler。



>在Java5.0及之后的版本中，可以通过Thread.setUncaughtExceptionHandler为每个线程设置一个UncaughtExceptionHandler，还可以使用setDefaultUncaughtExceptionHandler来设置默认的UncaughtExceptionHandler。然而，在这些异常处理器中，只有其中一个将被调用——JVM首先搜索每个线程的异常处理器，然后再搜索一个ThreadGroup的异常处理器。ThreadGroup中的默认异常处理器实现将异常处理工作逐层委托给它的上层ThreadGroup，直至其中某个ThreadGroup的异常处理器能够处理该未捕获异常，否则将一直传递到顶层的ThreadGroup。顶层ThreadGroup的异常处理器委托给默认的系统处理器（如果存在，在默认情况下为空），否则将把栈追踪信息输出到控制台。

~线程的异常处理器的调用顺序，依次是线程，线程组，系统默认的异常处理器。




#### 7.4.2　守护线程
>普通线程与守护线程之间的差异仅在于当线程退出时发生的操作。当一个线程退出时，JVM会检查其他正在运行的线程，如果这些线程都是守护线程，那么JVM会正常退出操作。当JVM停止时，所有仍然存在的守护线程都将被抛弃——既不会执行finally代码块，也不会执行回卷栈，而JVM只是直接退出。  
我们应尽可能少地使用守护线程——很少有操作能够在不进行清理的情况下被安全地抛弃。特别是，如果在守护线程中执行可能包含I/O操作的任务，那么将是一种危险的行为。守护线程最好用于执行“内部”任务，例如周期性地从内存的缓存中移除逾期的数据。  

~不要使用守护线程的理由，想想也是的，你不可能在没有关闭文件，关闭网络连接的情况下关闭JVM。
