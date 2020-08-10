---
layout: default
title: 第15章，原子变量与非阻塞同步机制
---

### 第15章，原子变量与非阻塞同步机制

#### 15.2.1　比较并交换
>CAS包含了3个操作数——需要读写的内存位置V、进行比较的值A和拟写入的新值B。当且仅当V的值等于A时，CAS才会通过原子方式用新值B来更新V的值，否则不会执行任何操作。无论位置V的值是否等于A，都将返回V原有的值。（这种变化形式被称为比较并设置，无论操作是否成功都会返回。）CAS的含义是：“我认为V的值应该为A，如果是，那么将V的值更新为B，否则不修改并告诉V的值实际为多少”。CAS是一项乐观的技术，它希望能成功地执行更新操作，并且如果有另一个线程在最近一次检查后更新了该变量，那么CAS能检测到这个错误。

~既然JDK中不少的使用了CAS，那么就有必要了解一下这个CAS（比较并交换）。


#### 15.2.2　非阻塞的计数器
>CAS的主要缺点是，它将使调用者处理竞争问题（通过重试、回退、放弃），而在锁中能自动处理竞争问题（线程在获得锁之前将一直阻塞）。

~任何一项技术都有它的问题，只需要将它用到最合适的地方即可。



#### 15.3.1　原子变量是一种“更好的volatile”#
>通过对指向不可变对象（其中保存了下界和上界）的引用进行原子更新以避免竞态条件。在CasNumberRange中使用了AtomicReference和IntPair来保存状态，并通过使用compareAndSet，使它在更新上界或下界时能避免NumberRange的竞态条件。  
```java
public class CasNumberRange {
	@Immutable
	private static class IntPair {
		final int lower; // Invariant: lower <= upper
		final int upper;
		...
	}
  //
	private final AtomicReference<IntPair> values =
		new AtomicReference<IntPair>(new IntPair(0, 0));
    //
		public int getLower() { return values.get().lower; }
		public int getUpper() { return values.get().upper; }
    //
		public void setLower(int i) {
			while (true) {
				IntPair oldv = values.get();
				if (i > oldv.upper)
					throw new IllegalArgumentException(
						"Can’t set lower to " + i + " > upper");
				IntPair newv = new IntPair(i, oldv.upper);
				if (values.compareAndSet(oldv, newv))
					return;
			}
		}
	// similarly for setUpper
}
```

~注意点： IntPair类中两个属性都是final的，另外就是通过 AtomicReference进行原子更新。通过组合不可变对象和引用的原子更新，就可以在操作多个变量时避免竞态条件的问题。


#### 15.4.4　ABA问题#
>如果在算法中采用自己的方式来管理节点对象的内存，那么可能出现ABA问题。在这种情况下，即使链表的头节点仍然指向之前观察到的节点，那么也不足以说明链表的内容没有发生改变。如果通过垃圾回收器来管理链表节点仍然无法避免ABA问题，那么还有一个相对简单的解决方案：不是更新某个引用的值，而是更新两个值，包括一个引用和一个版本号。即使这个值由A变为B，然后又变为A，版本号也将是不同的。AtomicStampedReference（以及AtomicMarkableReference）支持在两个变量上执行原子的条件更新。AtomicStampedReference将更新一个“对象-引用”二元组，通过在引用上加上“版本号”，从而避免ABA问题。类似地，AtomicMarkableReference将更新一个“对象引用-布尔值”二元组，在某些算法中将通过这种二元组使节点保存在链表中同时又将其标记为“已删除的节点”。
```java
AtomicStampedReference<Integer> account = new AtomicStampedReference<>(100, 0);
//
public boolean withdrawal(int funds) {
    int[] stamps = new int[1];
    int current = this.account.get(stamps);
    int newStamp = this.stamp.incrementAndGet();
    return this.account.compareAndSet(current, current - funds, stamps[0], newStamp);
}
//
public boolean deposit(int funds) {
    int[] stamps = new int[1];
    int current = this.account.get(stamps);
    int newStamp = this.stamp.incrementAndGet();
    return this.account.compareAndSet(current, current + funds, stamps[0], newStamp);
}
```

~解决ABA问题，这里提供了一个解决方案，另外JDK中提供相关的类。
