---
layout: default
title: 第5章，基础构建模块
---

## 第5章，基础构建模块

### 5.1.1　同步容器类的问题$
>同步容器类都是线程安全的，但在某些情况下可能需要额外的客户端加锁来保护复合操作。容器上常见的复合操作包括：迭代（反复访问元素，直到遍历完容器中所有元素）、跳转（根据指定顺序找到当前元素的下一个元素）以及条件运算，例如“若没有则添加”（检查在Map中是否存在键值K，如果没有，就加入二元组（K,V））。在同步容器类中，这些复合操作在没有客户端加锁的情况下仍然是线程安全的，但当其他线程并发地修改容器时，它们可能会表现出意料之外的行为。

~我现在的疑问就是迭代，跳转这个两个复合操作在不加锁的情况，会出现什么问题呢?我想主要在迭代或跳转时，如果正在处理的元素被其他线程替换或删除，可能会出现意料之外的行为，比如死锁等。


>我们可以通过在客户端加锁来解决不可靠迭代的问题，但要牺牲一些伸缩性。通过在迭代期间持有Vector的锁，可以防止其他线程在迭代期间修改Vector。然而，这同样会导致其他线程在迭代期间无法访问它，因此降低了并发性。

~注意的是持有Vertor的锁才行，而不是自己创建的锁。


### 5.1.3　隐藏迭代器
>容器的hashCode和equals等方法也会间接地执行迭代操作，当容器作为另一个容器的元素或键值时，就会出现这种情况。同样，containsAll、removeAll和retainAll等方法，以及把容器作为参数的构造函数，都会对容器进行迭代。所有这些间接的迭代操作都可能抛出ConcurrentModificationException。

~这些隐藏的迭代行为需要特别注意，或许就是并发问题的爆发点。

### 5.2.1 ConcurrentHashMap
>ConcurrentHashMap也是一个基于散列的Map，但它使用了一种完全不同的加锁策略来提供更高的并发性和伸缩性。ConcurrentHashMap并不是将每个方法都在同一个锁上同步并使得每次只能有一个线程访问容器，而是使用一种粒度更细的加锁机制来实现更大程度的共享，这种机制称为分段锁（LockStriping）。在这种机制中，任意数量的读取线程可以并发地访问Map，执行读取操作的线程和执行写入操作的线程可以并发地访问Map，并且一定数量的写入线程可以并发地修改Map。ConcurrentHashMap带来的结果是，在并发访问环境下将实现更高的吞吐量，而在单线程环境中只损失非常小的性能。

~ConcurrentHashMap高性能原因是因为使用了分段锁，使得任意多个线程可并发访问Map，而且一定数量的写线程可并发修改Map。


### 5.3　阻塞队列和生产者-消费者模式
>在类库中包含了BlockingQueue的多种实现，其中，LinkedBlockingQueue和ArrayBlocking-Queue是FIFO队列。PriorityBlockingQueue是一个按优先级排序的队列，当你希望按照某种顺序而不是FIFO来处理元素时，这个队列将非常有用。正如其他有序的容器一样，PriorityBlockingQueue既可以根据元素的自然顺序来比较元素（如果它们实现了Comparable方法），也可以使用Comparator来比较。  
>最后一个BlockingQueue实现是SynchronousQueue，实际上它不是一个真正的队列，因为它不会为队列中元素维护存储空间。与其他队列不同的是，它维护一组线程，这些线程在等待着把元素加入或移出队列。因为SynchronousQueue没有存储功能，因此put和take会一直阻塞，直到有另一个线程已经准备好参与到交付过程中。仅当有足够多的消费者，并且总是有一个消费者准备好获取交付的工作时，才适合使用同步队列。

~阻塞队列的几个实现，分别为FIFO队列，优先队列，同步队列。


### 5.3.3　双端队列与工作密取
>Java6增加了两种容器类型，Deque（发音为“deck”）和BlockingDeque，它们分别对Queue和BlockingQueue进行了扩展。Deque是一个双端队列，实现了在队列头和队列尾的高效插入和移除。具体实现包括ArrayDeque和LinkedBlockingDeque。

~双端队列，要知道有这个东西。

### 5.4  阻塞方法与中断方法?
>当在代码中调用了一个将抛出InterruptedException异常的方法时，你自己的方法也就变成了一个阻塞方法，并且必须要处理对中断的响应。对于库代码来说，有两种基本选择：  
传递InterruptedException。避开这个异常通常是最明智的策略——只需把InterruptedException传递给方法的调用者。传递InterruptedException的方法包括，根本不捕获该异常，或者捕获该异常，然后在执行某种简单的清理工作后再次抛出这个异常。  
恢复中断。有时候不能抛出InterruptedException，例如当代码是Runnable的一部分时。在这些情况下，必须捕获InterruptedException，并通过调用当前线程上的interrupt方法恢复中断状态，这样在调用栈中更高层的代码将看到引发了一个中断。  

~两种处理中断的方法，第一方法好理解，第二种方法，如果是将一个任务提交给线程池执行，那么是否适应这种方法呢？难道是说要将线程池中的执行此任务的线程设置为中断状态吗？好像也不合理，还是没有完全理解。


### 5.5.1　闭锁#
>闭锁是一种同步工具类，可以延迟线程的进度直到其到达终止状态。闭锁的作用相当于一扇门：在闭锁到达结束状态之前，这扇门一直是关闭的，并且没有任何线程能通过，当到达结束状态时，这扇门会打开并允许所有的线程通过。当闭锁到达结束状态后，将不会再改变状态，因此这扇门将永远保持保持打开状态。闭锁可以用来确保某些活动直到其他活动都完成后才继续执行，  
>例如：·确保某个计算在其需要的所有资源都被初始化之后才继续执行。二元闭锁（包括两个状态）可以用来表示“资源R已经被初始化”，而所有需要R的操作都必须先在这个闭锁上等待。  
>
>CountDownLatch类的Javadoc中的一个例子：
```java
class Driver { // ...
   void main() throws InterruptedException {
     CountDownLatch startSignal = new CountDownLatch(1);
     CountDownLatch doneSignal = new CountDownLatch(N);  
     for (int i = 0; i < N; ++i) // create and start threads
       new Thread(new Worker(startSignal, doneSignal)).start();
     doSomethingElse();            // don't let run yet
     startSignal.countDown();      // let all threads proceed
     doSomethingElse();
     doneSignal.await();           // wait for all to finish
   }
 }
 //
 class Worker implements Runnable {
   private final CountDownLatch startSignal;
   private final CountDownLatch doneSignal;
   Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
      this.startSignal = startSignal;
      this.doneSignal = doneSignal;
   }
   public void run() {
      try {
        startSignal.await();
        doWork();
        doneSignal.countDown();
      } catch (InterruptedException ex) {} // return;
   }
   //
   void doWork() { ... }
 }
```

~应该理解并记住。

### 5.5.2　FutureTask
>FutureTask也可以用做闭锁。（FutureTask实现了Future语义，表示一种抽象的可生成结果的计算）。FutureTask表示的计算是通过Callable来实现的，相当于一种可生成结果的Runnable，并且可以处于以下3种状态：等待运行（Waitingtorun），正在运行（Running）和运行完成（Completed）。“执行完成”表示计算的所有可能结束方式，包括正常结束、由于取消而结束和由于异常而结束等。当FutureTask进入完成状态后，它会永远停止在这个状态上。Future.get的行为取决于任务的状态。如果任务已经完成，那么get会立即返回结果，否则get将阻塞直到任务进入完成状态，然后返回结果或者抛出异常。FutureTask将计算结果从执行计算的线程传递到获取这个结果的线程，而FutureTask的规范确保了这种传递过程能实现结果的安全发布。

~当线程池执行完任务之后，就会根据执行的情况为FutureTask类的实例设置相关参数，比如状态，这个状态可以表示正常完成，任务撤销，还可以将任务执行过程当中的异常设置到FutureTask类的实例中，以便调用者线程调用get方法可以得到结果。  
~其实当程序将Callable任务提交给线程池时，就是将任务包装成FutureTask后提交给线程池去执行，然后返回这个对象的实例，调用程序就可以根据此对象的get方法来获取执行结果。


### 5.5.4	栅栏#
>栅栏（Barrier）类似于闭锁，它能阻塞一组线程直到某个事件发生。<u>栅栏与闭锁的关键区别在于，所有线程必须同时到达栅栏位置，才能继续执行。闭锁用于等待事件，而栅栏用于等待其他线程。</u>栅栏用于实现一些协议，例如几个家庭成员决定在某个地方集合：“所有人6：00在麦当劳碰头，到了以后要等其他人，之后再讨论下一步要做的事情。”    
>
>CyclicBarrier类的Javadoc中的一个伪代码例子：
```java
 class Solver {
   final int N;
   final float[][] data;
   final CyclicBarrier barrier;
   //
   class Worker implements Runnable {
     int myRow;
     Worker(int row) { myRow = row; }
     public void run() {
       while (!done()) {
         processRow(myRow);
         //
         try {
           barrier.await();
         } catch (InterruptedException ex) {
           return;
         } catch (BrokenBarrierException ex) {
           return;
         }
       }
     }
   }
   //
   public Solver(float[][] matrix) {
     data = matrix;
     N = matrix.length;
     barrier = new CyclicBarrier(N,
                                 new Runnable() {
                                   public void run() {
                                     mergeRows(...);
                                   }
                                 });
     for (int i = 0; i < N; ++i)
       new Thread(new Worker(i)).start();
       //
     waitUntilDone();
   }
 }
```

~栅栏与闭锁的关键区别前者等待的是线程，后者等待的是事件。栅栏中的线程调用await方法阻塞自己，直到所有的线程都已到达，线程才被唤醒继续执行。

### x.Exchanger
>另一种形式的栅栏是Exchanger，它是一种两方（Two-Party）栅栏，各方在栅栏位置上交换数据。当两方执行不对称的操作时，Exchanger会非常有用，例如当一个线程向缓冲区写入数据，而另一个线程从缓冲区中读取数据。

~可以了解一下，有这样一种形式的栅栏Exchanger。

### 5.6　构建高效且可伸缩的结果缓存
>将用于缓存值的Map重新定义为ConcurrentHashMap＜A,Future＜V＞＞，替换原来的ConcurrentHashMap＜A,V＞。首先检查某个相应的计算是否已经开始（与之相反，它首先判断某个计算是否已经完成）。如果还没有启动，那么就创建一个FutureTask，并注册到Map中，然后启动计算：如果已经启动，那么等待现有计算的结果。结果可能很快会得到，也可能还在运算过程中，但这对于Future.get的调用者来说是透明的。  
```java
public class Memoizer<A, V> implements Computable<A, V> {
	private final ConcurrentMap<A, Future<V>> cache
					= new ConcurrentHashMap<A, Future<V>>();
	private final Computable<A, V> c;
  //
	public Memoizer(Computable<A, V> c) { this.c = c; }
  //
	public V compute(final A arg) throws InterruptedException {
		while (true) {
			Future<V> f = cache.get(arg);
			if (f == null) {
				Callable<V> eval = new Callable<V>() {
					public V call() throws InterruptedException {
						return c.compute(arg);
					}
				};
				FutureTask<V> ft = new FutureTask<V>(eval);
				f = cache.putIfAbsent(arg, ft);//这一句很是关键，无值返回则运行。
				if (f == null) { f = ft; ft.run(); }
			}
			try {
				return f.get();
			} catch (CancellationException e) {
				cache.remove(arg, f);
			} catch (ExecutionException e) {
				throw launderThrowable(e.getCause());
			}
		}
	}
}
```

~这个方法用来解决某一线程执行某一任务时，其他的线程检查缓存找不到运行后的结果而后开始同一任务，导致任务运行多次的问题。解决的方法就是将Future对象存入ConcurrentHashMap 对象，这样当非第一个进入的线程可以检测到future已存在于Map中，防止线程再次计算。
