---
layout: default
---

# 第三部分，高效Java8编程（第8章到第12章）

## 第8章，重构、测试和调式

### 为改善可读性和灵活性重构代码

>* 重构代码，用Lambda表达式取代匿名类
>* 用方法引用重构Lambda表达式
>* 用Stream API重构命令式的数据处理

~Java8改善代码的可读性的方法。

>某些情况下，将匿名类转换为Lambda表达式可能是一个比较复杂的过程。首先，匿名类和Lambda表达式中的this和supper的含义是不同的。在匿名类中，this代表的是类自身，但是在Lambda中，它代表的是包含类。其次，匿名类可以屏蔽包含类的变量，而Lambda表达式不能，譬如下面的代码：
```java
int a = 10;
Runnable r1 = () -> {
  int a = 2;//编译错误
  System.out.println(a);
};
Runnable r2 = new Runnable(){
  int a = 2;//正常
  System.out.println(a);
};
```

~

**增加代码的灵活性**
>如果你发现你需要频繁地从客户端代码去查询一个对象的状态，只是为了传递参数、调用该对象的一个方法，那么可以考虑实现一个新的方法，以Lambda或方法表达式作为参数，新方法在检查完该对象的状态之后才调动原来的方法。你的代码会因此而变得更易读，封装性更好。
```java
public void log(Level level,Supplier<String> msgSupplier){
  if(logger.isLoggable(level)){
    log(level,msgSupplier.get());
  }
}
```

## 第9章，默认方法

>其一，Java8允许在接口内声明静态方法。其二，Java8引入了一个新功能，叫默认方法，通过默认方法你可以指定接口方法的默认实现。换句话说，接口能提供方法的具体实现。因此，实现接口的累如果不显式地提供该方法的具体实现，就会自动继续默认的实现。这种机制可以使你平滑地进行接口的优化和演进。

~什么是默认方法以及引入默认方法的原因。

### 不断演进的API

>二进制级的兼容性表示现有的二进制执行文件能无缝持续链接（包括验证、准备和解析）和运行。比如，为接口添加一个方法就是二进制级的兼容，这种方式下，如果新添加的方法不被调用，接口已经实现的方法可以继续运行，不会出现错误。
>
>简单来说，源代码级的兼容性表示输入变化后，现在的程序依然能成功编译通过。比如，向接口添加新的方法就不是源码级的兼容，因为遗留代码并没有实现新引入的方法，所以它们无法顺利通过编译。
>
>最后，函数行为的兼容性表示变更发生后，程序接受同样的输入能得到同样的结果。比如，为接口添加新的方法就是函数行为兼容的，因为新添加的方法在程序中并未被调用（抑或该接口在实现中被覆盖了）。

~什么是二进制兼容性，源代码级兼容性和函数行为的兼容性


### 概述默认方法
>那么抽象类和抽象接口之间的区别是什么呢？它们不都是包含抽象方法和包含方法体的实现吗？
>首先，一个类只能继承一个抽象类，但是一个类可以实现多个接口。
>其次，一个抽象类可以通过实例变量（字段）保存一个通用状态，而接口是不能有实例变量的。

~java8中的抽象类与抽象接口的区别。


### 解决冲突的规则

>如果一个类的默认方法使用相同的函数签名继承自多个接口，解决冲突的机制其实相当简单。你只需要遵守下面这三条准则就能解决所有可能的冲突。
>首先，类或父类中显式声明的方法，其优先级高于所有的默认方法。
>如果其第一条无法判断，方法签名又没有区别，那么选择提供最具体实现的默认方法的接口。
>最后，如果冲突依旧无法解决，你就只能在你的类中覆盖该默认方法，显式地指定在你的类中使用哪一个接口中的方法。
>显式地的调用父接口的方法，X.super.m(),其中X是你希望调用的m方法所在的父接口。

~解决冲突的规则


## 第10章，用Optional取代null

### Optional类入门
>在你的代码中始终如一地使用Optional，能非常清晰地界定出变量值的缺失是结构上的问题，还是你算法上的缺陷，亦或是你数据中的问题。<u>另外，我们还想特别强调，引入Optional类的意图并非要消除每一个null引用。与此相反，它的目标是帮助你更好地设计出普适的API，让程序员看到方法签名，就能了解它是否接受一个Optional的值。</u>这种强制会让你更积极地将变量从Optional中解包出来，直面缺失的变量值。

~引入Optional类的好处，我应该要知道。


### 应用Optional的几种模式

**创建Optional对象**
```java
//声明一个空的Optional
Optional<Car> optCar = Optional.empty();
//依据一个非空值创建Optional
Optional<Car> optCar = Optional.of(car);
//可接受null的Optional
Optional<Car> optCar = Optional.ofNullable(car);
```

~知道如何创建这个对象，就能够很好的使用它。


```java
//使用map从Optional对象中提取和转换值
//使用flatMap链接Optional对象
public String getCarInsuranceName(Optional<Person> person){
  return person.flatMap(Person::getCar)
                .flatMap(Car::getInsurance)
                .map(Insurance::getName)
                .orElse("Unknown");
}
```

~Optional中的flatMap，map方法的使用。

>我们再一次看到这种方法（将T改为Optional<T>）的优点，它通过类型系统让你的域模型中隐藏的知识显式地体现在你的代码中，<u>换句话说，你永远都不应该忘记语言的首要功能就是沟通，即使对程序设计语言而言也没有什么不同。声明方法接受一个Optional参数，或者将结果作为Optional类型返回，让你的同事或未来你方法的使用者，很清楚地知道它可以接受空值，或者它可能返回一个空值。</u>

~重要


## 第11章，CompletableFuture：组合式异步编程

### 实现异步API
>使用工厂方法supplyAsync创建CompletableFuture对象
```java
public Future<Double> getPriceAsync(String product){
  return CompletableFuture.supplyAsync(() -> calculatePrice(product));
}
```
>supplyAsync方法接受一个生产者（Supplier）作为参数，返回一个CompletableFuture对象，该对象完成异步执行后会读取调用生产者方法的返回值。生产者方法会交由ForkJoinPool池中的某个执行线程运行，但是你也可以使用supplyAsync方法的重载版本，传递第二个参数指定不同的执行线程执行生产者方法。

~需要了解

### 让你的代码免受阻塞之苦

>使用CompletableFuture实现findPrices方法
```java
public Future<Double> findPrices(String product){
  List<CompletableFuture<String>> priceFutures =
    shops.stream()
    .map(shop -> CompletableFuture.supplyAsync(
      () -> shop.getName() + " price is " +
      shop.getprice(product)))
    .collect(Collectors.toList);
  //
  return priceFutures.stream()
          .map(CompletableFuture::join)
          .collect(toList());
}
```
>这里使用了两个不同的Stream流水线，而不是在同一个处理流的流水线上一个接一个地放置两个map操作——这其实是有缘由的。考虑流操作之间的延迟特征，如果你在单一流水线中处理流，发向不同商家的请求只能以同步，顺序执行的方式才会成功。因此，每个创建CompletableFuture对象只能在前一个操作结束后执行查询指定商家的动作、通知Join方法返回计算结果。

~这个我也有点懵，不是说流可以并发执行吗？为什么不能在同一个流中设置两个map呢？什么是流的延迟特征？
~看了下面摘要，你就会明白，其实使用流也能够并行执行，只需要调用 parallel方法就行，这里使用CompletableFuture，因为它有更多的灵活性。

**使用定制的执行器**
>目前为止，你已经知道对集合进行并行计算有两种方式：要么将其转为为并行流，利用map这样的操作开展工作，要么枚举出集合中的每一个元素，创建新的线程，在CompletableFuture内对其进行操作。后者提供了更多的灵活性，你可以调整线程池的大小，而这能帮助你确保整体的计算不会因为线程都在等待I/O而发生阻塞。
>
>我们对使用这些API的建议如下。
>
>如果你进行的是计算密集型的操作，并且没有I/O，那么推荐使用Stream接口，因为实现简单，同时效率也可能是最高的（如果所有的线程都是计算密集型，那就没有必要创建比处理器核数更多的线程）。
>
><u>反之，如果你并行的工作单元还涉及等待I/O的操作（包括网络连接等待），那么使用CompletableFuture灵活性更好，你可以依据等待/计算，或者W/C的比例设定需要使用的线程数。这种情况不适用并行流的另一个原因是，处理流的流水线中如果发生I/O等待，流的延迟特征会让我们很难判断到底什么时候触发了等待。</u>

~这个很重要，让我知道何时使用Stream，何时使用CompletableFuture。

### 对多个异步任务进行流水线操作

使用CompletableFuture实现findPrices方法
```java
public Future<Double> findPrices(String product){
  List<CompletableFuture<String>> priceFutures =
    shops.stream()
    .map(shop -> CompletableFuture.supplyAsync(
      () -> shop.getPrice(product),executor)  //以异步方式取得每个shop中指定的产品的原始价格。
    .map(future -> future.thenApply(Quote::parse))  //Quote对象存在时，对其返回值进行转换。
    .map(future -> future.thenCompose(quote ->
          CompletableFuture.supplyAsync(
            () -> Discount.applyDiscount(quote),executor))) //使用另一个异步的任务构造期望的future申请折扣。
    .collect(Collectors.toList);

  return priceFutures.stream()
          .map(CompletableFuture::join)
          .collect(toList());   //等待流中的所有Future执行完毕，并提取各自的返回值。
}
```

~当做一个例子。



## 第12章，新的日期和时间API

### LocalDate、LocalTime、Instant、Durtion以及Period

>LocalDate例子
```java
LocalDate date = LocalDate.of(2019,12,6);
int year = date.getYear();
DayOfWeek dow = date.getDayOfWeek();
//
LocalDate today = LocalDate.now();
//
int year = date.get(ChronoField.YEAR);
```

>LocalTime例子
```java
LocalTime time = LocalTime.of(13,45,20);
int hour = time.getHour();
//
LocalDate date = LocalDate.parse("2019-12-06");
LocalTime time = LocalTime.parse("13:45:20");
```

>LocalDateTime
```java
LocalDateTime dt1 = LocalDateTime.of(2019,Month.MARCH,18,13,45,20);
LocalDateTime dt2 = LocalDateTime.of(date,Time);
LocalDateTime dt3 = date.setTime(13,45,20);
LocalDateTime dt4 = date.setTime(time);
LocalDateTime dt5 = date.setDate(date);
//
LocalDate date1 = dt1.toLocalDate();
LocalTime time1 = dt1.toLocalTime();
```

>Instant
```java
Instant.ofEpochSecond(3);
Instant.ofEpochSecond(2,1_000_000_000);//2秒之后再加上10亿纳秒（1秒）
```


>Duration Peroid
```java
Duration d1 = Duration.between(time1,time2);
Duration d2 = Duration.between(dateTime1,dateTime2);
Duration d3 = Duration.between(instant1,instant2);
//
Peroid tenDays = Peroid.between(LocalDate.of(2019,11,06),LocalDate.of(2019,11,16));
//
Duration threeMinutes = Duration.ofMinutes(3);
//
Peroid tenDays = Peroid.ofDays(10);
```

### 操纵、解析和格式化日期
```java
//以比较直观的方式操纵LocalDate的属性
LocalDate date1 = LocalDate.of(2019,12,6);
LocalDate date2 = date1.withYear(2018);//2018-12-6
LocalDate date3 = date1.withDayOfMonth(25);//2019-12-25
LocalDate date4 = date1.with(ChronoField.MONTH_OF_YEAR,9);//2019-9-6
//以相对方式修改LocalDate对象的属性
LocalDate date5 = date1.plusWeeks(1);//2019-12-13
LocalDate date6 = date1.minusYears(3);//2016-12-6
LocalDate date7 = date1.plus(6,ChronoUnit.MONTHS);
```

**使用TemporalAdjuster**
>有的时候，你需要进行一些更加复杂的操作，比如，将日期调整到下一周日，下个工作日，或者是本月的最后一天。这时，你可以使用重载版本的with方法，想其传递一个提供了更多定制化选择的TemporalAdjuster对象，更加灵活地处理日期。
```java
LocalDate date1 = LocalDate.of(2014,3,18);
LocalDate date2 = date1.with(nextOrSame(DayOfWeek.SUNDAY));//2014-03-23
LocalDate date3 = date2.with(lastDayOfMonth());//2014-03-31
```

**打印输出及解析日期-时间对象**
```java
LocalDate date = LocalDate.of(2014,3,18);
String s1 = date.format(DateTimeFormatter.BASIC_ISO_DATE);//20140318
String s2 = date.format(DateTimeFormatter.ISO_LOCAL_DATE);//2014-03-18
//
LocalDate date1 = LocalDate.parse("20140318",DateTimeFormatter.BASIC_ISO_DATE);
LocalDate date2 = LocalDate.parse("2014-03-18",DateTimeFormatter.ISO_LOCAL_DATE);
//
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy");
String formatterDate = date1.format(formatter);
LocalDate date3 = LocalDate.parse(formatterDate,formatter);
```
