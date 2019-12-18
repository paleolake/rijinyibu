---
layout: default
---

# 第四部分，超越Java8（第13章到第16章）

## 第13章，函数式的思考

### 什么是函数式编程

**函数式Java编程**

>我们的准则是，被称为“函数式”的函数或方法都只能修改本地变量。除此此外，它引用的对象都应该是不可修改的对象。通过这种规定，我们期望所有的字段都为final类型，所有的引用类型字段都指向不可变对象。
>我们前述的准则是不完备的，要成为真正的函数式程序还有一个附加条件，不过它在最初时不太为大家所重视。要被称为函数式，函数或方法不应该抛出任何异常。
>那么，如果不使用异常，你该如何对除法这样的函数进行建模呢？答案是请使用Optional<T>类型：你应该避免让sqrt使用double sqrt(double)这样的函数签名，因为这种方式可能抛出异常；与之相反我们推荐你使用Optional\<Double\> sqrt(double)——这种方式下，函数要么返回一个值表示调用成功，要么返回一个对象，表明其无法进行指定的操作。
><u>最后，作为函数式的程序，你的函数或方法调用的库函数如果有副作用，你必须设法隐藏它们的非函数式行为，否则就不能调用这些方法（换句话说，你需要确保它们对数据结构的任何修改对于调用者都是不可见的，你可以通过首次复制，或者捕获任何可能抛出的异常实现这一目的）。</u>

~函数式的函数或方法的准则，应该有所了解。


### 递归和迭代

>使用Java8进行编程，我们有一个建议，你应该尽量使用Stream取代迭代操作，从而避免变化带来的影响。此外，如果递归能让你以更精炼，并且不带任何副作用的方式实现算法，你就应该用递归替换迭代。
```java
//迭代式的阶乘计算
static int factorialIterative(int n){
  int r = 1;
  for(int i = 1; i <= n; i++){
    r *= i;
  }
  return r;
}
//递归式的阶乘计算
static long factorialRecursive(long n){
  return n == 1 ? 1 : n * factorialRecursive(n - 1);
}
//基于Stream的阶乘
static long factorialStream(long n){
  return LongStream.rangeClosed(1,n)
                    .reduce(1,(long a,long b) -> a * b);
}
```

~递归与迭代的选择。


## 第14章，函数式编程的技巧

### 无处不在的函数

**科里化**

>科里化的理论定义。科里化是一种将具备2个参数（比如，x和y）的函数f转化为使用一个参数的函数g，并且这个函数的返回值也是一个函数，它会作为新函数的一个参数。后者的返回值和初始函数的返回值相同，即f(x,y)=(g(x))(y)。
```java
static double converter(double x,double f,double b){
  return x * f + b;
}
//科里化
static DoubleUnaryOperator curriedConverter(double f,double b){
  return (double x) -> x * f + b;
}
//调用
DoubleUnaryOperator converterUSDtoGBP = curriedConverter(0.6,0);
double gbp = converterUSDtoGBP.applyAsDouble(1000);
```
>你并没有一次性地向converter方法传递所有的参数x，f和b，相反，你只是使用了参数f和b并返回了另一个方法，这个方法会接受参数x，最终返回你期望的值x*f+b。通过这种方式，你复用了现有的转换逻辑，同时又为不同的转换因子创建了不同的转换方法。

~科里化以及相关的实例。
