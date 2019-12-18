---
layout: default
---

# 第一部分，基础知识（第1章到第3章）

## 第2章，通过行为参数化传递代码

>行为参数化就是可以帮助你处理频繁变更的需求的一种软件开发模式。一言以蔽之，它意味着拿出一个代码块，把它准备好却不去执行它。这个代码块以后可以被你的程序的其他部分调用，这意味着你可以推迟这块代码的执行。例如，你可以将代码块作为参数传递给另一个方法，稍后再去执行它。这样，这个方法的行为就基于那块代码被参数化了。

~行为参数化？

## 第3章，Lambda表达式

### Lambda管中窥豹

>你可以把Lambda表达式理解为简洁地表示可传递的匿名函数的一种方式：它没有名称，但它有参数列表、函数主体、返回类型，可能还有一个可以抛出的异常列表。
>
>有三个部分：参数列表，箭头，Lambda主体。
```java
(Apple a1,Apple a2) -> a1.getWeight().compareTo(a2.getWeight());
```

~什么是Lambda表达式以及Lambda表达式的写法。

### 在哪里以及如何使用Lambda

**函数式接口**
>函数式接口就是只定义一个抽象方法的接口。
>
>用函数式接口可以干什么呢？Lambda表达式允许你直接以内联的形式为函数式接口的抽象方法提供实现，并把整个表达式作为函数式接口的实例（具体说来，是函数式接口一个具体实现的实例）。

~什么是函数式接口及其用处。

### 使用函数式接口

>java.util.function.Predicate<T>接口定义了一个名叫test的抽象方法，它接受泛型T对象，并返回一个boolean。在你需要表示一个涉及类型T的布尔表达式时，就可以使用这个接口。
```java
Predicate<String> nonEmptyStringPredicate
                          = (String s) -> !s.isEmpty();
```
>java.util.function.Consumer<T>定义了一个名叫accept的抽象方法，它接受泛型T的对象，没有返回（void）。你如果需要访问类型T的对象，并对其执行某些操作，就可以使用这个接口。
```java
forEach(
  Arrays.asList(1,2,3,4,5),
  (Integer i) -> System.out.println(i)
);
```
>
>java.util.function.Function<T,R>接口定义了一个叫做apply的方法，它接受一个泛型T的对象，并返回一个泛型R的对象。如果你需要定义一个Lambda，将输入对象的信息映射到输出，就可以使用这个接口。
```java
List<Integer> l = map(
  Arrays.asList("lambdas","in","action")
  (String s) -> s.length
);
```

~三个常用的函数式接口的用法
>
>使用案例 | Lambda例子 | 对应的函数式接口
------------ | ------------- | -------------
布尔表达式  | (List<String> list) -> list.isEmpty() | Predicate<List<String>>
创建对象 | () -> new Apple(10) | Supplier<Apple>
消费一个对象 | (Apple a) -> System.out.println(e.getWeight())    | Consumer<Apple>  
从一个对象中选择/提取 | (String s) -> s.length() |  Function<String,Integer>或ToIntFunction<String>
合并两个值 | (int a, int b) -> a * b  |  IntBinaryOperator
比较两个对象 | (Apple a1, Apple a2) -> |  a1.getWeight.compareTo(a2.getWeight)     Comparator<Apple>或BiFunction<Apple,Apple,Integer>或ToIntBiFunction<Apple,Apple>

~Lambdas及函数式接口的例子


### 类型检查、类型推断以及限制

**使用局部变量**
>Lambda可以没有限制地捕获（也就是在其主体中引用）实例变量和静态变量。<u>但局部变量必须显式声明为final或事实上是final。换句话说，Lambda表达式只能捕获指派给它们的局部变量一次</u>。（注：捕获实例变量可以被看作捕获最终局部变量this。）
>
>如前所述，<u>这种限制存在的原因在于局部变量保存在栈上，并且隐式表示它们仅限于其所在线程。如果允许捕获可改变的局部变量，就会引起造成线程不安全的新的可能性，而这是我们不想看到的</u>（实例变量可以，因为它们保存在堆中，而堆是在线程之间共享的）。

~Lambda捕获局部变量的限制以及有此限制的原因。


### 方法引用
>它是如何工作的呢？当你需要使用方法引用时，目标引用放在分隔符::前，方法名称放在后面。例如Apple::getWeight就是引用了Apple类中定义的方法getWeight。请记住，不需要括号，因为你没有实际调用这个方法。方法引用就是Lambda表达式(Apple a) -> a.getWeight()的快捷写法。
```java
Integer::parseInt //静态方法的方法引用
String::length  //任意类型实例方法的方法引用
expensiveTransaction::getValue  //现有对象的实例方法的方法引用
```

~方法引用的写法。

**构造函数引用**
>对于一个现有构造函数，你可以利用它的名称和关键字new来创建它的引用：ClassName::new。它的功能与指向静态方法的引用类似。
```java
//无参的构造函数
Supplier<Apple> c1 = Apple::new;
Apple a1 = c1.get();
//一个参数的构造函数
Function<Integer,Apple> c2 = Apple::new;
Apple a2 = c2.apply(110);
//两个参数的构造函数
BiFunction<String,Integer,Apple> c3 = Apple::new;
Apple a3 = c3.apply("green",110);
```

~需要有所了解。


### 复合Lambda表达式的有用方法

**比较器复合**
```java
inventory.sort(comparing(Apple::getWeight)
              .reversed()
              .thenComparing(Apple::getCountry));
```

~先按重量降序排序，当两个苹果一样重时，再按国家排序。

**谓词复合**
谓词接口包括三个方法：negate、and和or，让你可以重用已有的Predicate来创建更复杂的谓词。
```java
  redApple.negate();//现有对象的非

  redApple.and(a -> a.getWeight() > 150);//链接两个谓词

  redApple.and(a -> a.getWeight() > 150)
           .or(a -> "green".equals(a.getColor()));//链接多个谓词。
```

~

**函数复合**
>最后，你还可以把Function接口所代表的Lambda表达式复合起来。Function接口为此配了andThen和compose两个默认方法，它们都会返回Function的一个实例。
>
>andThen方法会返回一个函数，它先对输入应用一个给定函数，再对输出应用另一个函数。
```java
  Function<Integer,Integer> f = x -> x + 1;
  Function<Integer,Integer> g = x -> x * 1;
  Function<Integer,Integer> h = f.andThen(g);
  int result = h.apply(1);
```

~两个复合函数，作用都一样。
