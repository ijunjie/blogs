---
title: "浅谈函数式编程—下篇：函数式编程基础"
date: 2018-04-21
draft: false
tags:
- fp
categories:
- fp
---

本篇是《浅谈函数式编程》的下篇，讲解一些重要的函数式编程概念。作为背景知识，阅读本篇前建议先了解 [浅谈函数式编程-上篇：编程范式]()

本篇选取了一些常见的函数式编程概念，使用多种语言的代码做示例。读者可从这些示例中体会函数式编程的核心思想和常用技巧。

导读：阅读本篇前，可以先思考如下问题：

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

函数式编程可以看做声明式编程的一个子集。近年来，命令式编程语言越来越多地引入函数式（声明式）编程特性。函数式编程已然成为主流编程语言的标配。

实际上很多人已经在不同程度上使用函数式编程了，如很多场合常见的map, filter，reduce,. 其中体现的是与命令式截然不同的思想：不使用选择和循环（看不到if和for），而选择和循环是结构化编程的两个核心。

不同语言的函数式能力可能有很大差别。一门语言是不是函数式语言，其标准是有争议的，往往会陷入无休止的争论。我们可以简单理解为一种程度上的差别。重要的是要领会和理解函数式思维方式。

函数式编程的概念纷繁复杂，但核心思想只有一点：不变、无副作用、无状态。各种概念都是围绕着这个核心思想的，而且各个概念之间互相有着深刻的联系。函数式的观点认为，一个函数的返回值是表达计算结果的唯一形式（而不是把结果写入参数或借助副作用输出）。当然这是理论上的，实际上计算机必须与人交互，计算机总是需要IO与人类交互。正因如此，一些函数式编程语言费劲各种心机处理函数式编程并不擅长的IO. 某种意义上讲其带来的麻烦远远多于便利。从实用主义角度看，能解决问题就是价值。

下面我们开始了解一些经典的函数式编程概念。

## 三板斧

上面提到的map, filter, reduce可以说是函数式编程的三板斧。也是很多初学函数式编程最先认识的三个函数。即使仅仅学会这三个函数，就足以是代码整洁度提高一个档次。

实际上，在Android开发或者响应式编程，如RxJava/RxJS中，这三个函数式很常见的。

Javascript也是支持这三个函数的。

map 和 filter:

```javascript
let a = [1, 2, 3, 4];
let r1 = a.map(e => e * e);
console.log(r1); // [1, 4, 9, 16]

let r2 = r1.filter(e => e % 2 == 0);
console.log(r2); // [4, 16]
```

reduce:

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

reduce在很多语言中都有支持。掌握其原理就会一通百通。

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

### Collector

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

## 一等公民

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

## 纯函数
## 副作用
## 闭包
## flatmap和monad
## FP不是银弹
