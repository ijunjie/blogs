---
title: "浅谈函数式编程—下篇：核心概念"
date: 2018-04-26
draft: false
tags:
- fp
categories:
- fp
---

本篇是《浅谈函数式编程》的下篇，主要讲解一些重要的函数式编程概念。上篇主要介绍编程范式： [浅谈函数式编程-上篇：编程范式](https://ijunjie.github.io/post/fp/fp-paradigm/)


导读：阅读本篇前，请思考如下问题：

1. 函数式编程中如何从形式上消除结构化编程所强调的选择和循环？
2. map, filter, reduce 如何实现整洁代码？
3. 为什么说在函数式编程中，代码和数据之间没有明确的界限？
4. 什么是lambda？不同的具有函数式编程特征的语言是如何支持lambda的？
5. 什么是纯函数？什么是副作用？
6. 什么是高阶函数，什么是柯里化？柯里化有什么意义？
7. 什么是闭包？
8. flatmap和monad有什么联系？
9. 为什么说函数式编程不是银弹？


## 函数式编程

函数式编程可以看做声明式编程的一个子集。近年来，命令式编程语言越来越多地支持声明式（函数式）的编程特性。支持函数式编程已然成为主流编程语言的标配。

实际上很多人已经在使用函数式编程了，如很多场合常见的map, filter，reduce. 其主要的特征体现的是与命令式截然不同的思想：不使用选择和循环（看不到if和for），而选择和循环是结构化编程的两个核心。

不同语言对函数式编程的支持能力有很大差别。一门语言是不是完全的函数式语言，其实并没有明确的界限。

函数式编程的核心思想是：不变、无副作用、无状态。各种概念都是建立在这个基础上的，概念之间环环相扣、互相有着深刻的联系。函数式编程认为，一个函数的返回值是表达计算结果的唯一形式（而不是把结果写入参数或借助副作用输出）。而实际上计算机最终必须通过IO与人交互。 正因如此，一些函数式编程语言做了很多工作来处理IO，由此也产生了不少难题。

本文要讨论的一些一些常见的、经典的函数式编程概念：

- map, reduce, filter
- first-class member
- lambda
- high-order function，currying
- pure function, side-effect
- closure
- flatmap, functor, monad

## 三板斧

上面提到的map, filter, reduce可以说是函数式编程的三板斧。也是很多初学函数式编程最先接触的三个函数。这三个函数很有用，即使仅仅学会这三个函数，也足以将代码整洁度提高一个档次。

在Android开发或者响应式编程，如RxJava/RxJS中，这三个函数是很常见的。

下面通过Javascript来看一下三个函数的用法：

### map 和 filter

```javascript
let a = [1, 2, 3, 4];
let r1 = a.map(e => e * e);
console.log(r1); // [1, 4, 9, 16]

let r2 = r1.filter(e => e % 2 == 0);
console.log(r2); // [4, 16]
```

### reduce

在对一个集合中的元素按顺序两两操作时，根据某种策略得到一个结果，得到的结果参与下一次操作，最终被归约为一个结果，即reduce的返回值。

有时需要提供一个初始值，同第一个元素进行运算，返回的值同第二个元素进行运算，以此类推。

```javascript
let a = [1, 2, 3, 4];
let r3 = a.reduce((s, e) => s + e, 0);
console.log(r3); // 10
```

返回值可以是与几何元素不同的类型：

```javascript
let a = [1, 2, 3, 4];
let r3 = a.reduce((s, e) => s.concat(e), "");
console.log(r3); // 1234
```

reduce在很多语言中都有支持。只要掌握其原理，语言不是问题。

Java 8:

```java
List<String> names = Arrays.asList("hello", "world", "functional");
String longestName = names.stream
        .reduce("program", (u, e) -> u.length() >= e.length() ? u: e);
Assert.assertTrue(longestName.equals("functional"));
```

reduce是Java 8的Stream的方法：

```java
<U> U reduce(U identity, BiFunction<U, T, U> accumulator);
```

### collector

实际上在Java 8不会经常用直接用到reduce. 更常用的是Java 8提供的工具类Collectors.

下面的方法将一个Map翻转：

```java
static <K, V> Map<V, Set<K>> reverse(Map<K, V> map) {
    if (map.values().contains(null))
        throw new IllegalArgumentException("Null value contained.");

    return map.entrySet().stream()
            .collect(Collectors.groupingBy(Map.Entry::getValue,
                Collectors.mapping(Map.Entry::getKey, Collectors.toSet())));
}
```

下面看一个词频统计的例子：

传统命令式风格：

```java
public Map<String, Long> frequency1() {
    TreeMap<String, Long> wordMap = new TreeMap<>();
    Matcher m = Pattern.compile("\\w+").matcher(words);
    while (m.find()) {
        String word = m.group().toLowerCase();
        if (!NON_WORDS.contains(word)) {
            Long count = wordMap.get(word);
            if (count == null) {
                wordMap.put(word, 1L);
            } else {
                wordMap.put(word, count + 1L);
            }
        }
    }
    return wordMap;
}
```

这段代码的特点：

- 不同任务**交织（complect）**在一起，用一次循环完成多个任务
- 每次迭代中有三项操作：转换为小写，滤除虚词，计算词频
- 牺牲了代码的清晰，换取执行性能
- 包含大量if判断，使用循环————遵循了结构化编程原则。


下面我们使用函数式编程风格重构。

实际上计算词频的几行代码可以使用`wordMap.merge(word, 1L, (a, b) -> a + b)`代替。但仅仅是这一点不够彻底。完全的函数式风格要像下面这样：

```java
public Map<String, Long> frequency2() {
    reurn wordStream(words, "\\w+")
            .map(String::toLowerCase)
            .filter(w ->!NON_WORDS.contains(w))
            .collect(
                Collectors.groupingBy(
                    Function.identity,
                    TreeMap::new,
                    Collectors.counting()));
}

private static Stream<String> wordStream(String words, String regex) {
    Stream.Builder<String> builder = String.builder();
    Matcher matcher = Pattern.compile(regex).matcher(words);
    while (matcher.find()) {
        builder.add(matcher.group());
    }
    return builder.build();
}
```

抽取出滤除虚词的函数wordStream. 由于Matcher和Pattern本身的原因，wordStream的实现无法重构成函数式风格。


可以看到，重构后的frequency2方法有以下特点：

- 使用map完成转换小写
- 使用filter完成虚词过滤
- 使用groupingBy完成词频统计
- 代码清晰
- 消除了显示的判断和循环
- 抽象层次得到提高

## first-class member

函数式编程语言中，函数通常作为一等公民（first-class member）. 其含义是，函数跟其他数据类型处于平等的位置。

- 函数可以赋值给变量
- 函数可以作为参数
- 函数可以作为返回值

有些OOP语言中，函数并不是一等公民（如Ruby），但仍然支持函数式编程风格。这跟基于对象有很大关系。如Java的Function，Ruby的method可以提取方法作为对象。

## lambda

lambda是函数式编程的核心。简单讲，lambda就是**匿名函数**。

不同语言对lambda的支持形式不同，约束也不同。关于函数、lambda和闭包的区别，请参考闭包一节。

下面是一些语言中的lambda形式：

Javascript:

```javascript

let f = i => console.log(i);

```

Java:

```java
Consumer<String> f = i -> System.out.println(i);
```

Scala:

```scala
val f: String => Unit = i => println(i)
```

Ruby:

```ruby
f = lambda { |i|
  put i
}
```

Groovy:

```groovy
def f1 = { i -> println(i)}
def f2 = { println it}
```

Python:

```python
f = lambda i: print(i)
```

## 高阶函数和柯里化

高阶函数简单来讲就是**接受函数作为参数**或**返回函数**的函数。

柯里化的函数具有多个参数列表。

“柯里”来自于著名数学家**Haskell B. Curry**. 没错，就是编程语言Haskell的Haskell。他的名字和姓氏分别成为了一门编程语言的名字和一个函数式概念的名字。

来看一下柯里化：

Javascript正常写法，调用时传递多个参数：

```javascript
let f1 = (a, b) => a + b;
let r1 = f1(1, 2)；
```

柯里化：

```javascript
let f2 = a => b => a + b;
let r2 = f2(1)(2);
```

可以看到，通过高阶函数可以实现拆分参数列表。实际上每个参数列表对应返回的是一个Partial Application.

柯里化是一种常用的函数式编程技巧。在一些语言中也可以用于辅助类型推断。

下面我们看一个高阶函数和递归综合运用的例子。

### 综合示例

下面我们通过一个javascript例子，一步步完成匿名函数的递归调用，充分展现高阶函数、递归、柯里化、IIFE的使用。

**函数自调**

```javascript
let a = f => f(f);
let r = a(function(f1) {
  return 2;
});
console.log(r);
```

这段代码首先定义了一个匿名函数a, a接收一个函数f，并用这个函数调用自身f(f). 也就是说这个f本身的参数也是一个函数。

接下来向a传参。这里传入的参数是`function(f1) {return 2;}`. 为了区分这里没有使用箭头函数。

a拿到这个参数后，执行f(f)自调，意味着f1又是`function(f1) {return 2}`. 但实际在函数体中并没有用到f1，而是直接返回了2.我们加入一行日志，查看一下：

```javascript
let a = f => f(f);
let r = a(function(f1) {
  console.log(f1.toString());
  return 2;
});
console.log(r)
```

Javascript的传参方式是很灵活的，多传参数不会有影响：

```javascript
let a = f => f(f);
let r = a(function(f1) {
  console.log(f1.toString());
  return 2;
}, "hello", "world", "foo", "bar");
console.log(r)
```

额外传递的参数会被a忽略。

接下来我们对a进行一些改造，使a接收的函数可以附带参数。

**带参数的函数自调**

```javascript
let a = (f, n) => f(f, n);
let r = a((f1, n1) => {
  console.log(n1);
  return 2;
}, 1);
console.log(r);
```

在这段代码中，1首先作为a的第二个参数传给n，然后实际又传给了f的第二个参数。f和a具有相同的结构。

接下来进行柯里化改造：

**柯里化**

```javascript
let a = f => n => f(f)(n);
let r = a(f1 => n1 => {
  console.log(n1);
  return 2;
})(1);
```

可以看到，通过高阶函数将a和f都拆分成多个参数列表。

**IIFE**

IIFE是 `Immediately Invoked Function Expression` 的缩写。用于一次性匿名函数的调用。

上面的代码可以省去中间变量a.

```javascript
let r = (f => n => f(f)(n))(f1 => n1 => 2)(1);
```

**最大公约数示例**

```javascript
let gcd = (x, y) => (f => a => b => f(f)(a)(b))
  (f => a => b => b == 0 ? a : f(f)(b)(a % b))
  (x)
  (y);
console.log(gcd(66, 42));
```
这个示例需要在较高版本的Firefox或Chrome运行.

## 纯函数和副作用

所谓纯函数是指，返回的结果仅仅依赖于传入的参数，不依赖外部变量和全局状态。

副作用：函数式思想认为，一个函数唯一的与所在环境交流的方式就是获得参数和返回结果。除此之外产生的改变都是副作用。

副作用是从函数的视角来看的，仅仅是叫做“副作用”。有些情况下，特别是命令式语言，涉及到数据结构和IO处理时，需要的恰恰是副作用，返回值当做了副产品。这是不符合函数式原则的。

### 引用透明

与纯函数和副作用密切相关的一个概念是引用透明。引用透明是指，表达式（引用）和值可以相互替换，无副作用，不影响函数结果。

这里参考了**雪川大虫**的介绍。雪川大虫有一系列关于泛函编程和Scalaz的文章，是学习Scala的非常有价值的资源。

Scala:

```scala
var x = "Hello World"
var r1 = x.reverse
var r2 = x.reverse // r1和r2中的x都可以替换为"Hello World"
```


```scala
var y = new StringBuilder("Hello")
var z = y.append(", World")

var r3 = z.toString()
var r4 = z.toString()
var r5 = z.toString()
```
这段代码中，r3, r4, r5中的z是不能替换为y.append(", World")的。因为append不是一个引用透明的操作。


## Function及其变体

基于纯函数的思想，对参数、返回值和副作用进行搭配组合，可以得到一些变体。从实用角度看这些变体能满足各种实际需求。

这里采用Java 8中的概念。实际上其他语言也大致通用。

|概念|参数|返回值|副作用|备注|
|:---:|:---:|:---:|:---:|:---:|
|Function|有|有|无|函数式编程的核心|
|Predicate|有|Boolean|无|固定返回值为Boolean类型|
|Supplier|无|有|无|不通过参数提供返回值|
|Consumer|有|无|有|对参数进行处理，需要的是副作用|
|Runnable|无|无|有|运行某种任务的过程；<br>（实际的参数在对象上下文中）|

下面的代码都是以Java 8代码结合OOP来讲的。由于在Java中函数不是一类对象，对函数式编程的支持是用类型和对象实现的。对开发者而言，这样做更容易接受。如在Stream上用`.`即可调用map, filter, 而在python中函数式和OOP结合的不好，map仍然是个函数而不是方法。

### Function

- Function<T, R>, T为参数类型， R为返回值类型；jdk也提供了BiFunction<T, U, R>, T和U为参数类型
- 方法：apply
- identity方法：返回自身 x -> x

```java
Function<Integer, String> f1 = i -> String.valueOf(i);
Function<Integer, String> f2 = String::valueOf; // method reference

String r1 = f1.apply(2);

Function<String, String> returnSelf = Function.identity();
```

### Predicate

- Predicate<T>, T为参数类型，返回值为Boolean，可以看做Function<T, Boolean>
- 方法：test
- BiPredicate
- 通常作为filter的参数

```java
Predicate<String> pred1 = s -> s != null;
BiPredicate<String, Integer> pred2 = (s, i) -> true;
boolean test1 = pred1.test("hello");
```

### Supplier, Consumer, Runnable

- Supplier<T>, 无参数，返回值为T类型，可看做Function<Void, T>, 方法get
- Consumer<T>, 无返回值，参数类型为T，可看做Function<T, Void>, 方法accept
- Runnable无参数，无返回值

```java
Supplier<String> supplier = () -> "hello";
String r = supplier.get();

Consumer<String> consumer = System.out::println;
consumer.accept("hello");

Runnable runnable = () -> {};
```
jdk同样也提供了以Bi开头的函数接口，如BiConsumer等，可以避免用户自行封装类型。

## 闭包

闭包是指，一个函数在**运行时**， 捕获**自由变量**并关闭。这个运行时的函数对象叫做闭包。

常见的闭包通常是嵌套在外部作用域中的私有函数。

自由变量和绑定变量：在函数的上下文中，有明确意义的变量叫做**绑定变量**，函数文本不能给出明确意义的变量叫做**自由变量**。

严格意义上的闭包是包含了**自由变量**的函数。这一点是区分闭包与（纯）函数的关键。

下面的表格展示了函数、lambda和闭包的区别。

|-|Function|Lambda|Closure|
|---|---|---|---|
|Named|Named/Anonymous|Anonymous|Named/Anonymous|
|Have free variables|Yes/No|Yes/No|Yes|


从这个表中可以看出，lambda区别与Function的关键点是匿名；closure区别于Function的关键点是包含自由变量；closure也是一种lambda.

（BTW，如果拿Method与它们三个比较，则Method是OOP的概念，它们三个是FP的概念。Method带有对象上下文，要处理this.）

Javascript闭包示例：

```javascript
var outer = () => {
  var a = [];
  for(var i = 0; i<5; i++) {
    a[i] = () => console.log(i);
  }
  return a[2];
};

outer()();
```

这段代码中，() => console.log(i);是一个lambda形式的闭包，因为i是自由变量。这个闭包只有在真正运行的时候才会去捕获i的值。因此最后outer()返回了该闭包，执行闭包时i已经变成5了。

假定我们这里希望输出的是1，那么如何修改？思路很简单，只要尽早获取i的值就可以了。

如： `a[i] = new Function("console.log(" + i +");")`

这句代码的关键是拼接字符串的表达式。表达式的计算是优先的，先于Function的构造。所以Function的构造中已经获取到了期望的i的值。

当然，除了用表达式之外，也有别的方式，如IIFE，采用最新的let语法等。

### 闭包对自由变量的读写

在很多语言中，自由变量的值发生变化后，闭包是可以感知到的。

Javascript：

```javascript
var more = 8;
var f = x => x + more;
console.log(f(3));
more = 9;
console.log(f(3));
```
有些语言中，闭包可以修改自由变量，如Javascript；有些语言中自由变量在闭包中是只读的，如Java.

```javascript
var sum = 0;
var arr = [1, 2, 3, 4];
arr.forEach(e => sum += e);
console.log(sum);
```

```java
String result = ""; // 要求effectively final
Optional.of("hello").ifPresent(s -> result = s); //编译不通过
```

### 闭包的意义和局限性

- 闭包延长了自由变量的声明周期，利用这一点可以用闭包模拟对象，实现对象机制
- 间接保持对象的值会给垃圾回收带来难题，也会降低性能
- 闭包使函数具有了保存数据的能力，但又不足以实现完善的面向对象能力

## flatmap和monad

这一节难度较高，如果只是想简单了解一下函数式编程，这一节可以略去不看。

flatmap可以看做flat + map, 但这并不是问题的本质。下面我们从类型的角度来看一下flatmap.

先看flatmap的用法：

```java
Stream<List<Integer>> listStream =
    Arrays.asList(Arrays.asList(1, 2), Arrays.asList(3, 4)).stream();
Stream<Integer> integerStream = listStream.flatMap(lst -> lst.stream());
integerStream.forEach(System.out::println);
```
上面这段代码将嵌套的list拉平了，没有体现map. 修改一下：

```java
Stream<String> stringStream = listStream.flatMap(lst -> lst.stream().map(i -> "[" + i + "]"));
```

可以看到 flatmap的确是flat + map的过程。

### 从类型角度分析flatmap

从类型角度来看：

> Stream<List<Integer>> 转换为 Stream<String>, flatMap的参数是 从List<Integer> 到 Stream<String> 的函数

从`List<Integer>`的流变成`Integer的流`体现了flat，但从类型上看`List<Integer>`和`String`是两个毫无关系的类型。我们把`Stream`抽象成Type，把`List<Integer>`抽象成T，把`String`抽象成U，可以得到

> Type<T> 转换为 `Type<U>`, flatMap的参数是 从 T 到 `Type<U>` 的函数

是的，这就是flatmap的本质。如果转换的方式不是从 `T` 到 `Type<U>`, 而是从 `T` 到 `U`， 就成了 map.

用类型来描述flatmap：

- 是`Type<T>`的方法
- 如果看做Function，则类型描述为 `Function<Function<T, Type<U>>, Type<U>>`
- 从OOP角度看，flatmap的第一个参数是this, 所以flatmap的完整形式应该是`Type<U> flatMap(Type<T> self, Function<T, Type<U>> f)`

下面我们来证明CompletionStage的thenCompose方法也是flatmap:

准备两种类型 T 和 U:

```java
static class T{}
static class U{}
```

```java
Stream<T> st = Stream.of(new T());
Function<Function<T, Stream<U>>, Stream<U>> flatmap1 = st::flatMap;

Function<T, U> t2u = t -> new U();
CompletionStage<T> ct = CompletableFuture.completedFuture(new T());

Function<Function<T, CompletionStage<U>>, CompletionStage<U>> alsoFlatmap = ct::thenCompose;
```

最后一句证明了thenCompose实际也是flatmap.

### functor和monad

- 一个functor(函子)是一种表示为`Type<T>`的类型。它封装了另一种类型，具有`(T->U) -> Type<U>`签名的map方法
- 一个monad(单子)是一种类型，它是一个functor，同时具有`(T->Type<U>) -> Type<U>`签名的flatmap方法

这是简明扼要的描述。实际monad的理论比较复杂。monad是为了处理IO副作用而出现的。

{{< blockquote >}} 谈论monad的第一原则是不要谈论monad；第二原则是不要读别人的关于monad的文章;第三原则是不要发表关于monad的文章误导他人。  {{< /blockquote >}}


## FP不是银弹

最后，我们用一句带有道德训诫意味话做本篇的收尾——FP不是银弹。

银弹在计算机领域常用来指代某种策略、技术或技巧，可以极大解放程序员的生产力。

《没有银弹》是Fred Brooks在1987年发表的一片关于软件工程的经典论文。论文中强调真正的银弹并不存在。没有任何一项技术或方法可以让软件工程的生产力在十年内提高十倍。这篇论文的核心思想通常被解释为**复杂的软件工程问题无法靠简单的答案来解决**。

函数式编程的局限性：

- 需要副作用的操作：流程控制、顺序操作、状态维护、IO运算
- 表现力、执行效率问题。函数式编程风格接近于数学运算，没有受过训练的程序员难以理解表意密度极高的代码；执行效率上一般来说也不如命令式，因为计算机操作系统底层归根结底还是命令式的。


函数式编程的用武之地：

- 并行计算、并发模型
- 大数据处理
- 人工智能

人类社会创造的一切事物都在朝着更加智能的方向进化。声明式、函数式、逻辑式编程是面向未来的编程思想，把更多工作交给机器去做，注重提高人的效率，而不是机器的效率。

（完）
