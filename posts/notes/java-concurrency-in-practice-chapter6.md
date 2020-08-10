---
layout: default
title: 第6章，任务执行
---

### 第6章，任务执行

#### 6.2.3　线程池
>可以通过调用Executors中的静态工厂方法之一来创建一个线程池：  
newFixedThreadPool。newFixedThreadPool将创建一个固定长度的线程池，每当提交一个任务时就创建一个线程，直到达到线程池的最大数量，这时线程池的规模将不再变化（如果某个线程由于发生了未预期的Exception而结束，那么线程池会补充一个新的线程）。  
newCachedThreadPool。newCachedThreadPool将创建一个可缓存的线程池，如果线程池的当前规模超过了处理需求时，那么将回收空闲的线程，而当需求增加时，则可以添加新的线程，线程池的规模不存在任何限制。  
newSingleThreadExecutor。newSingleThreadExecutor是一个单线程的Executor，它创建单个工作者线程来执行任务，如果这个线程异常结束，会创建另一个线程来替代。newSingleThreadExecutor能确保依照任务在队列中的顺序来串行执行（例如FIFO、LIFO、优先级）。  
newScheduledThreadPool。newScheduledThreadPool创建了一个固定长度的线程池，而且以延迟或定时的方式来执行任务，类似于Timer。  

~我将newScheduledThreadPool称为调度线程池。对于通过Executors中的静态工厂方法创建的线程池，还是有必要了解一下。


>单线程的Executor还提供了大量的内部同步机制，从而确保了任务执行的任何内存写入操作对于后续任务来说都是可见的。这意味着，即使这个线程会不时地被另一个线程替代，但对象总是可以安全地封闭在“任务线程”中。  

~难道说这种线程池会在线程异常退出时，会将异常退出的线程中的缓存及时的刷新到内存，防止后续新创建的线程不能获取正确的内存值的问题。对于同一个线程执行多个任务，任务之间应该不存在这个问题，因为在同一个线程中执行，所以前一个任务的执行结果对后一个任务都是可见的。



#### 6.2.4　Executor的生命周期
>ExecutorService的生命周期有3种状态：运行、关闭和已终止。ExecutorService在初始创建时处于运行状态。shutdown方法将执行平缓的关闭过程：不再接受新的任务，同时等待已经提交的任务执行完成—包括那些还未开始执行的任务。shutdownNow方法将执行粗暴的关闭过程：它将尝试取消所有运行中的任务，并且不再启动队列中尚未开始执行的任务。

~应该要知道这两种关闭方法的区别，说不定在以后的工作会用到。



#### 6.2.5　延迟任务与周期任务$
>如果要构建自己的调度服务，那么可以使用DelayQueue，它实现了BlockingQueue，并为ScheduledThreadPoolExecutor提供调度功能。DelayQueue管理着一组Delayed对象。每个Delayed对象都有一个相应的延迟时间：在DelayQueue中，只有某个元素逾期后，才能从DelayQueue中执行take操作。从DelayQueue中返回的对象将根据它们的延迟时间进行排序。

~其中第一句对照英语原文，我的理解它实现了BlockingQueue，并提供了ScheduledThreadPoolExecutor的调度功能，这样理解，可能才是对的。简单来说就是DelayQueue用来管理着Delayed对象，只有过了延迟时间，才能从此队列中取出。


#### 6.3.2　携带结果的任务Callable与Future
>有些任务可能要执行很长的时间，因此通常希望能够取消这些任务。在Executor框架中，已提交但尚未开始的任务可以取消，但对于那些已经开始执行的任务，只有当它们能响应中断时，才能取消。取消一个已经完成的任务不会有任何影响。  
>Future.get方法的行为取决于任务的状态（尚未开始、正在运行、已完成）。如果任务已经完成，那么get会立即返回或者抛出一个Exception，如果任务没有完成，那么get将阻塞并直到任务完成。如果任务抛出了异常，那么get将该异常封装为ExecutionException并重新抛出。如果任务被取消，那么get将抛出CancellationException。如果get抛出了ExecutionException，那么可以通过getCause来获得被封装的初始异常。

~了解一下。



#### 6.3.5　CompletionService：Executor与BlockingQueue$
>如果向Executor提交了一组计算任务，并且希望在计算完成后获得结果，那么可以保留与每个任务关联的Future，然后反复使用get方法，同时将参数timeout指定为0，从而通过轮询来判断任务是否完成。这种方法虽然可行，但却有些繁琐。幸运的是，还有一种更好的方法：完成服务（CompletionService）。  
CompletionService将Executor和BlockingQueue的功能融合在一起。你可以将Callable任务提交给它来执行，然后使用类似于队列操作的take和poll等方法来获得已完成的结果，而这些结果会在完成时将被封装为Future。ExecutorCompletionService实现了CompletionService，并将计算部分委托给一个Executor。  

~对于CompletionService来说，take和poll方法只能获取已完成的结果的Future，如果任务都没有完成，take方法将一直阻塞，而poll方法可以设置阻塞的时间。  
~如何知道要执行几次take或poll方法呢？难道放在一个循环中，还是在调用此方法时有感知？


#### 6.3.8　示例：旅行预定门户网站
>限时的invokeAll方法将多个任务提交到一个ExecutorService并获得结果。InvokeAll方法的参数为一组任务，并返回一组Future。这两个集合有着相同的结构。invokeAll按照任务集合中迭代器的顺序将所有的Future添加到返回的集合中，从而使调用者能将各个Future与其表示的Callable关联起来。当所有任务都执行完毕时，或者调用线程被中断时，又或者超过指定时限时，invokeAll将返回。当超过指定时限后，任何还未完成的任务都会取消。当invokeAll返回后，每个任务要么正常地完成，要么被取消，而客户端代码可以调用get或isCancelled来判断究竟是何种情况。

~invokeAll方法可以通过方法参数设置执行超过时间，可以用于用户点击取消按钮中断，比如浏览器输入某一个网址后回车，但是一段时间后没有响应，用户点击取消访问，还比如在规定的时间内获取所有渠道的消息，或在一定的时间内进行某项操作等。
