---
layout: default
title: 第4章，对象的组合
---

### 第4章，对象的组合

#### 4.1　设计线程安全的类

>在设计线程安全类的过程中，需要包含以下三个基本要素：·找出构成对象状态的所有变量。·找出约束状态变量的不变性条件。·建立对象状态的并发访问管理策略。

~我理解不变性条件应该就是变量的取值约束，比如性别就只能是男和女，如果多个变量参与到一个不变性条件，例如，太阳系的行星以及行星到太阳的平均距离，现在已知的只有9大行星，你就不能创建出另外一个行星出来，此外你也不能将冥王星，地球两个行星到太阳的平均距离进行交换，这些约束就是不变性条件。


#### 4.2.1　Java监视器模式

>使用私有的锁对象而不是对象的内置锁（或任何其他可通过公有方式访问的锁），有许多优点。私有的锁对象可以将锁封装起来，使客户代码无法得到锁，但客户代码可以通过公有方法来访问锁，以便（正确或者不正确地）参与到它的同步策略中。如果客户代码错误地获得了另一个对象的锁，那么可能会产生活跃性问题。此外，要想验证某个公有访问的锁在程序中是否被正确地使用，则需要检查整个程序，而不是单个的类。

~私有的锁对象就是在类中创建一个私有的变量，然后利用这个私有变量的内置锁。以前知道这两种写法，但是不知道私有的锁对象有这样的好处。

#### 4.4.2　组合

>当为现有的类添加一个原子操作时，有一种更好的方法：组合（Composition）。

```java
@ThreadSafe
public class ImprovedList<T> implements List<T> {
	private final List<T> list;

	public ImprovedList(List<T> list) { this.list = list; }

	public synchronized boolean putIfAbsent(T x) {
		boolean contains = list.contains(x);
		if (contains)
			list.add(x);
		return !contains;
	}

	public synchronized void clear() { list.clear(); }
	// ... similarly delegate other List methods
}
```

>ImprovedList通过自身的内置锁增加了一层额外的加锁。它并不关心底层的List是否是线程安全的，即使List不是线程安全的或者修改了它的加锁实现，ImprovedList也会提供一致的加锁机制来实现线程安全性。虽然额外的同步层可能导致轻微的性能损失，但与模拟另一个对象的加锁策略相比，ImprovedList更为健壮。事实上，我们使用了Java监视器模式来封装现有的List，并且只要在类中拥有指向底层List的唯一外部引用，就能确保线程安全性。

~第一句已经进行了说明，后面的例子只是用于更好的说明如何做，知道有这种方法很好。


#### 4.5　将同步策略文档化

>更糟糕的是，我们的直觉通常是错误的：我们认为“可能是线程安全“的类通常并不是线程安全的。例如，java.text.SimpleDateFormat并不是线程安全的，但JDKl.4之前的Javadoc并没有提到这点。许多开发人员都对这个类不是线程安全的而感到惊讶。

~说实话，我也不知道SimpleDateFormat不是线程安全，都将它作为实例变量，这样使用方式就有可能在大并发量的情况出现错误。