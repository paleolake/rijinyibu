---
layout: default
title: 第3章，对象的共享
---

### 第3章，对象的共享

#### 3.1.4 Volatile变量

>加锁机制既可以确保可见性又可以确保原子性，而volatile变量只能确保可见性。
>当且仅当满足以下所有条件时，才应该使用volatile变量：  
>1.对变量的写入操作不依赖变量的当前值，或者你能确保只有单个线程更新变量的值。  
>2.该变量不会与其他状态变量一起纳入不变性条件中。  
>3.在访问变量时不需要加锁。

~我认为下面的理解是错误的，不变性条件是指多个变量要一起变化，就想转账，转款人的账上金额需要减去转账金额，而收款人需要加上此金额。如果只是转款人账上金额减少了，而收款人却没有增加，或者相反，都是不正确的。也就是说多个变量需要同时发生变化，并达到一个正确的其他状态。
~其中不变性条件是什么，我的理解变量只能在变量的取值范围内取值，比如订单状态，可以分为未付款，已付款，已发货，已收货等，那么就只能这几个状态内取值，且状态变更顺序也有限制，不能从已付款变更为未付款等。


#### 3.3.3　ThreadLocal类

>在实现应用程序框架时大量使用了ThreadLocal。例如，在EJB调用期间，J2EE容器需要将一个事务上下文（TransactionContext）与某个执行中的线程关联起来。通过将事务上下文保存在静态的ThreadLocal对象中，可以很容易地实现这个功能：当框架代码需要判断当前运行的是哪一个事务时，只需从这个ThreadLocal对象中读取事务上下文。这种机制很方便，因为它避免了在调用每个方法时都要传递执行上下文信息，然而这也将使用该机制的代码与框架耦合在一起。  
>开发人员经常滥用ThreadLocal，例如将所有全局变量都作为ThreadLocal对象或者作为一种“隐藏”方法参数的手段。ThreadLocal变量类似于全局变量，它能降低代码的可重用性，并在类之间引入隐含的耦合性，因此在使用时要格外小心。  

~线程相关的值都保存在Thread对象中，当线程终止后，这些值也将作为垃圾回收掉。这是因为在Thread类中有一个类Hashmap对象，每次通过ThreadLocal对象可以为当前线程设置一个属性值（如果想要设置多个属性值，可以通过列表，集合等），也就是说一个ThreadLocal只能为线程设置一个属性值，因为将属性值存放到Thread的哈希结构中时是以ThreadLocal对象的实例为键，属性值为值存放的。当然可以通过多个ThreadLocal对象的实例为键存放多个值。另外一点就是可以通过同一个ThreadLocal对象为多个线程设置属性。
~第二个需要注意点ThreadLocal是为了方便某个值或某个对象在线程中传递而创建的，并不是为了值或对象在线程间的共享，所以一般都是线程运行的run方法中设置某个值或某个对象，然后在线程的方法调用过程中共享这个值或对象。