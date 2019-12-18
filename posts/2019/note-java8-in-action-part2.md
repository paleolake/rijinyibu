---
layout: default
---

# 第二部分，函数式数据处理（第4章到第7章）

## 第4章，引入流

### 流简介
>流到底是什么呢？简短的定义就是“从支持数据处理操作的源生成的元素序列”。流操作有两个重要的特点：流水线，内部迭代。

~流是什么，工作中需要用到它，那么就应该知道它是什么。

### 流与集合
>粗略地说，集合与流之间的差异就在于什么时候进行计算。<u>集合是一个内存中的数据结构，它包含数据结构中目前所有的值——集合中的每个元素都得先算出来才能添加到集合中。</u>
>相比之下，<u>流则是在概念上固定的数据结构（你不能添加或删除元素），其元素则是按需计算的。</u>这对编程有很大的好处。这个思想就是用户仅仅从流中提取需要的值，而这些值——在用户看不见的地方——只会按需生成。这是一种生产者-消费者的关系。从另一角度来说，流就像是一个延迟创建的集合：只有在消费者要求的时候才会计算值。

~流与集合的区别。


### 流操作

**使用流**
>总而言之，流的使用一般包括三件事：
>1、一个数据源（如集合）来执行一个查询；
2、一个中间操作链，形成一条流的流水线；
3、一个终端操作，执行流水线，并能生成结果。

~知道就行。

## 第5章，使用流

### 筛选和切片
>1、谓词筛选
```java
  menu.stream()
      .filter(Dish::isVegetarian)//
      .collect(toList());
```

>2、筛选各异的元素
```java
  numbers.stream()
          .filter(i -> i % 2 == 0)
          .distinct()//
          .forEach(System.out::println);
```

>3、截断流
```java
  menu.stream()
      .filter(d -> d.getCalories() > 300)
      .limit(3)//
      .collect(toList());
```

>4、跳过元素
```java
  menu.stream()
      .filter(d -> d.getCalories() > 300)
      .skip(2)//
      .collect(toList());
```

~筛选和切片例子，记录在这里，便于以后使用。

### 映射
>对流中每一个元素应用函数
```java
  menu.stream()
      .map(Dish::getName)//
      .map(String::length)//
      .collect(toList());
```

~列举的例子，方便以后使用。

**流的扁平化**
```java
  words.stream()
      .map(w -> w.split(""))
      .flatMap(Arrays::stream)//
      .distinct()
      .collect(toList());
```
><u>使用flatMap方法的效果是，各个数组并不是分别映射成一个流，而是映射成流的内容。所有使用map(Arrays::stream)时生成的单个流都被合并起来，即扁平化为一个流。</u>
>
>一言以蔽之，flatMap方法让你把一个流中的每个值都换成另一个流，然后把所有的流连接起来成为一个流。

~将各个生成流扁平化为单个流。


### 查找与匹配
Stream API通过allMatch、anyMatch、noneMatch、findMatch和findAny方法来处理数据集中的某些元素是否匹配一个给定的属性。
```java
  if(menu.stream().anyMatch(Dish::isVegetarian)){}

  Optional<Dish> dish =
                    menu.stream()
                        .filter(Dish::isVegetarian)
                        .findAny();
```

~例子

### 归约

**元素求和**

>reduce接受两个参数：
1、一个初始值，这里是0；
2、一个BinaryOperator<T>来将两个元素结合起来产生一个新值。
```java
  int sum = numbers.stream().reduce(0,(a,b) -> a+b);
```

~记录在这里，


**最大值和最小值**
```java
  Optional<Integer> max = numbers.stream().reduce(Integer::max);

  Optional<Integer> min = numbers.stream().reduce(Integer::min);
```

~例子，便于以后直接参考。

### 数值流

**映射到数值流**
```java
  int calories = menu.stream()
                      .mapToInt(Dish::getCalories)//
                      .sum();
```

**转换回对象流**
```java
  IntStream intStream = menu.stream()
                          .mapToInt(Dish::getCalories);
  Stream<Integer> stream = intStream.boxed();//
```

**数值范围**
```java
  IntStream evenNumbers = IntStream
                      .rangeClosed(1,100)//range(1,100)
                      .filter(n -> n % 2 == 0);
```

### 构建流
>1、由值构建流
```java
  Stream<String> stream = Stream.of("Lambdas","in","action");
  Stream<String> emptyStream = Stream.empty();
```
>2、由数组创建流
```java
  int[] numbers = {1,2,3,4,5};
  int sum = Stream.stream(numbers).sum();
```
>3、由文件生成流
```java
  Stream<String> lines =
   Files.lines(Paths.get("data.txt"),Charset.defaultCharset());
```
>4、由函数生成流：创建无限流
>由iterate和generate产生的流会用给定的函数按需创建值，因此可以无穷无尽地计算下去。一般来说，应该使用limit来对这种流加以限制，以避免打印无穷多个值。
```java
  Stream.iterate(0,n->n+2)
        .limit(10)
        .forEach(System.out::println);
```
>与iterate方法类似，generate方法也可以让你按需生成一个无限流。<u>但generate不是依次对每个新生成的值应用函数的。它接受一个Supplier<T>类型的Lambda提供新的值。</u>``
```java
  Stream.generate(Math::random)
        .limit(10)
        .forEach(System.out::println);
```

## 第6章，用流收集数据

### 规约和汇总

>查询流中的最大值和最小值
```java
  Comparator<Dish> dishColoriesComparator =
      Comparator.comparingInt(Dish::getCalories);//比较器
  Optional<Dish> mostCalorieDish =
      menu.stream()
          .collect(maxBy(dishColoriesComparator));
```

>汇总
```java
  //平均值
  double avgCalories =
      menu.stream().collect(averagingInt(Dish::getCalories));
  //统计值：数量，总和，平均值，最大值，最小值。
  IntSummaryStatices menuStatistics =
      menu.stream().collect(summarizingInt(Dish::getCalories));
```

>连接字符串
```java
  String shortMenu =
   menu.stream().map(Dish::getName).collect(joining(", "));
```

**广义的归约汇总**
>事实上，我们已经讨论的所有收集器，都是一个可以用reducing工厂方法定义的归约过程的特殊情况而已。Collectors.reducing工厂方法是所有这些特殊情况的一般化。
```java
  int totalCalories =
     menu.stream()
     .collect(reducing(0,Dish::getCalories,(i,j) -> i+j));
```
>它有三个参数：
>第一个参数是归约操作的起始值，也是流中没有元素时的返回值，所以很显然对于数值和而言0是一个合适的值。
>第二个参数就是一个函数。
>第三个参数是一个BinaryOperator，将两个项目累积成一个同类型的值。

~知道这个有帮助。

**收集与归约**
>语义问题在于，reduce方法旨在把两个值结合起来生成一个新值，它是一个不可变的归约。与此相反，collect方法的设计就是要改变容器，从而累积要输出的结果。以错误的语义使用reduce方法还会造成一个实际问题：这个归约过程不能并行工作，因为由多个线程并发修改同一个数据结构可能会破坏List本身。在这种情况下，如果你想要线程安全，就需要每次分配一个新的List，而对象分配又会影响性能。这就是collect方法特别适合表达可变容器上的归约的原因，更关键的是它合适并行操作。

~大多数的时候都要考虑使用collect方法，只有在求和，求最大值，最小值，平均值可以考虑使用reduce方法。

### 分组
```java
  Map<Dish.type,List<Dish>> dishesByType =
    menu.stream().collect(
          groupingBy(Dish::getType)
        );
```

**多级分组**
```java
  Map<Dish.type,Map<CaloricLevel,List<Dish>>> dishesByTypeCaloricLevel =
    menu.stream().collect(
            groupingBy(Dish::getType,
              groupingBy(dish -> {
                if(dish.getCalories() <= 400 return CaloricLevel.DIET);
                else if(dish.getCalories() <= 700) return CaloricLevel.NORMAL;
                else return CaloricLevel.FAT;
              })
            )
          );
```

**按子组收集数据**
>传递给第一个groupingBy的第二个参数可以是任何类型，而不一定是另一个groupingBy，如下，可以传递counting收集器作为groupingBy收集器的第二个参数。
```java
  Map<Dish.type,Long> typesCount =
    menu.stream().collect(
          groupingBy(Dish::getType,counting())//
        );
```
>把收集器的结果转换为另一种类型
```java
  Map<Dish.type,Dish> mostCaloricByType =
    menu.stream().collect(
          groupingBy(Dish::getType,
            collectingAndThen(
              maxBy(comparingInt(Dish::getCalories)),
              Optional::get
            ))
        );
```

### 分区

>分区是分组的特殊情况：由一个谓词（返回一个布尔值的函数）作为分类函数，它称为分区函数。分区函数返回一个布尔值，这意味着得到的分组Map的键类型是Boolean，于是它最多可以分为两组——true是一组，false是一组。
```java
Map<Boolean,List<Dish>> partitionedMenu =
            menu.stream().collect(
              partitioningBy(Dish::isVegetarian));
```


## 第7章，并行数据处理与性能

### 并行流
>在现实中，对顺序流调用parallel方法并不意味着流本身有任何实际的变化。它在内部实际上就是设了一个boolean标志，表示你想让调用parallel之后进行的所有操作都并行执行。类似地，你只需要对并行流调用sequential方法就可以把它变成顺序流。
```java
  Stream.iterate(1L,i -> i + 1)
        .limit(n)//n=1000000
        .parallel()
        .reduce(0L,Long::sum);
```

**高效使用并行流**
>高效使用并行流的建议：
>* 如果有疑问，测量。并行流并不总是比顺序流快。此外，并行流有时候会和你的直觉不一致，所以在考虑选择顺序流还是并行流时，第一个也是最重要的建议就是用适当的基准来检查性能。
>* 留意装箱。自动装箱和拆箱操作会大大减低性能。Java8中有原始类型流（IntStream，LongStream，DoubleStream）来避免这种操作，但凡有可能都应该用这些流。
>* 有些操作本身在并行流上的性能就比顺序流差。特别是limit和findFirst等依赖于元素顺序的操作，它们在并行流上执行的代价非常大。
>* 要考虑流背后的数据结构是否易于分解。例如，ArrayList的拆分效率比ListedList高得多，因为前者用不着遍历就可以平均拆分，而后者则必须遍历。


### 分支/合并框架

**使用RecursiveTask**
>要把任务提交到这个池，必须创建RecursiveTask<R>的一个子类，其中R是并行化任务（以及所有子任务）产生的结果类型，或者如果任务不返回结果，则是RecursiveAction类型（当然它可能会更新其他非局部机构）。要定义RecursiveTask，只需实现它唯一的抽象方法compute：
```java
  protected abstract R compute();
```
>这个方法同时定义了将任务拆分成子任务的逻辑，以及无法再拆分或不方便在拆分时，生成单个子任务结果的逻辑。正由于此，这个方法的实现类似于下面的伪代码：
```java
  if(任务足够小或不可分){
    顺序计算该任务
  } else {
    将任务分成两个任务
    递归调用本方法，拆分每个子任务，等待所有子任务完成
    合并每个子任务的结果
  }
```

~任务拆分

**使用分支/合并框架的最佳做法**
>* 对于一个任务调用join方法会阻塞调用方，直到该任务做出结果。<u>因此，有必要在两个子任务的计算都开始之后再调用它。否则，你得到的版本会比原始的顺序算法更慢更复杂，因为每个子任务都必须等待另一个子任务完成才能启动。</u>
>* 不应该在RecursiveTask内部使用ForkJoinPool的invoke方法。相反，你应该始终直接调用compute或fork方法，只有顺序代码才应该用invoke来启动并行计算。
>* 对子任务调用fork方法可以把它排进ForkJoinPool。<u>同时对左边和右边的子任务调用它似乎很自然，但这样做的效率要比直接对其中一个调用compute低。这样做你可为其中一个子任务重用同一线程，从而避免再线程池中多分配一个任务造成的开销。</u>

~使用ForkJoinPool时，这几点很重要。


**工作窃取**

>不幸的是，实际中，每个子任务所花的时间可能天差万别，要么是因为划分策略效率低，要么是有不可预知的原因，比如磁盘访问慢，或者需要和外部服务协调执行。
>
>分支/合并框架工程用一种称为工作窃取的技术来解决这个问题。在实际应用中，这意味着这些任务差不多被平均分配到ForkJoinPool中的所有线程上。每个线程都为分配给它的任务保存一个双向链式队列，每完成一个任务，就会从队列头上取出下一个任务开始执行。<u>基于前面所述的原因，某个线程可能早早完成了分配给它的所有任务，也就是它的队列已经空了，而其他线程还很忙。这时，这个线程并没有闲下来，而是随机选择一个别的线程，从队列的尾巴上“偷走”一个任务。这个过程一直继续下去，直到所有的任务都执行完毕，所有的队列都清空。这就是为什么要划成许多小任务而不是少数几个大任务，这有助于更好地在工作线程之间平衡负载。</u>

~需要了解。
