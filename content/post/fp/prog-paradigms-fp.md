---
title: "编程范式和函数式编程"
date: 2018-04-23
draft: false
tags:
- fp
categories:
- fp
---

这篇长文是改编自我在公司内部的一次培训PPT. 编程范式部分主要参考了**郑晖**的《冒号课堂：编程范式与OOP思想》，电子工业出版社。这本书气象宏大，体例独特，有些观点可能具有争议性，但不可否认它是一本极富启发性的好书。本文中关于编程范式的一些论述，基本采用了这本书中观点。

## 当我们谈论函数式编程时，我们在谈论什么？

> map, flatmap, closure,filter, reduce, functor, monoid, monad, currying, lambda, cps, predicate...

(请自动脑补标签云图...)

阅读本文前，需要思考的一些问题：

1. 编程范式是什么，三大核心编程范式有哪些？
2. 第四代、第五代编程语言有什么特点？第三代语言的演变趋势是什么？
3. 命令式编程和过程式编程有什么联系？结构化编程是什么？
4. 声明式编程和函数式编程有什么关系？
5. 什么是响应式编程？
6. 函数式编程如何应用于数据处理、并发编程和响应式编程？
7. 函数式编程和面向对象是对立的吗？
8. 什么是逻辑式编程，应用在哪些领域？
9. 为什么说从长远来看，人的效率比机器的效率重要？
10. 函数式编程中如何消除显示的 for 循环？
11. map, filter, reduce 如何通过抽象层次的提高实现整洁代码？
12. 为什么说在函数式编程中，代码和数据之间没有明确的界限？
13. 什么是lambda？不同的具有函数式编程特征的语言是如何支持lambda的？
14. 什么是纯函数？什么是副作用？
15. 什么是高阶函数，什么是柯里化？柯里化有什么意义？
16. 什么是闭包？
17. flatmap和monad有什么联系？
18. 为什么说函数式编程不是银弹？

大家阅读完本文后，可以对编程范式和函数式编程有基本的了解，扩展自己在编程范式和编程语言方面的视野，进而能够对自己的**编码方式**和**抽象思维层次**有所启发。


## 编程范式

### 编程范式的地位

编程范式的地位经常受到忽视。在IT世界中，数据结构和算法、操作系统、网络、数据库、分布式系统、架构与设计无疑具有非常重要的地位。要在这些领域有所成就，离不开大量的编程实践。而在编程实践活动中，程序员总是自觉或不自觉地采用某种世界观和方法论，这种世界观和方法论实际就是**编程范式**。

编程范式是编程的心法，体现了一个人的思维方式；不同编程范式在不同语言中体现为**编程风格**的不同。

### 编程范式的定义

范式译自因为的 `paradigm`, 也译作“典范”、“范型”、“范例”。

> 编程范式（programming paradigm）指的是计算机编程的基本风格或典范模式

可以说，编程范式是构建虚拟世界的世界观和方法论。

### 编程范式和编程语言

编程范式和编程语言具有如下关系：

- 抽象和具体。编程范式是抽象的，必须在编程语言中才能体现。
- 多对多对应关系：一种范式可以见于多种语言，一种语言也可以支持多种范式
- 主导范式：一种语言通常具有一种主导范式，由此形成这门语言的风格特征

## 编程语言

迄今为止，编程语言的演变大致经历了5代：

1. 机器语言
2. 汇编语言
3. 高级语言
4. 面向特定领域问题的语言
5. 人工智能语言

目前主流的编程语言集中在第三、四、五代。

第三代语言是目前世界上最主要、最流行的编程语言，我们见到的大多数高级语言都是第三代；

第四代语言专注于业务逻辑和问题领域，**不是通用性的**，不满足[图灵完备](https://en.wikipedia.org/wiki/Turing_completeness).

第五代语言在保持了第三代语言的通用性，此外一个突出的特点是面向**人工智能**领域。第四代和第五代语言有很多共同点，**强调目标而不是过程，强调描述而不是实现**。

实际上，第三代语言也在朝着这个方向进化。近年来很多第三代语言的新版本中增加了很多支持声明式、函数式的特性。

下面举几个例子说明一下**目标**和**过程**的区别。

### 例一

SQL语言属于第四代语言，注重目标，而不是过程。如：

```sql
SELECT customer, SUM(order_price) FROM orders
GROUP BY customer
HAVING SUM(order_price) < 2000
```
使用SQL语言的人只需要声明自己想要的结果即可，而不必关心获取数据的具体过程。

### 例二

这个例子来源于《Java 8 函数式编程》，Richard Warburton著。

使用 Java 实现在 windows 控制台调用 help, 代码如下：

```java
try {
    Process proc = Runtime.getRuntime().exec("help");
    BufferedReader result = new BufferedReader(new InputStreamReader(proc.getInputStream(),
        "GBK"));
    String line;
    while ((line = result.readLine) != null) {
        System.out.println(line);
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

这段代码体现了 Java 语言设计中根深蒂固的命令式思想。代码编写者不需要关心如此多的细节。

作为对比，使用 Groovy 实现如下：

```groovy
println "help".execute().text
```

接下来，我们分别了解一下主要的编程范式：命令式、声明式、逻辑式。

## 命令式编程

命令式编程是**最常见、最传统、最普及**的编程范式。

为什么？因为绝大多数高级语言都是汇编语言的升级，汇编语言是从机器语言发展而来的，而机器语言大多是在**冯诺依曼机**发展起来的。冯诺依曼机是基于命令式的：从机器内存中**依序**获取指令和数据并执行。所以说，**命令式编程本质上是冯诺依曼机器的运行机制的抽象**。

命令式编程的核心观点是：程序是由若干指令组成的有序列表。

命令式编程的方法论：用变量存储数据，用语句执行命令。

BTW，有没有非命令式的机器语言？理论上是有的。只能存在于非冯诺依曼机上，如数据流机、归约机。

### 过程式编程

过程式编程是指引入了procedure、function和subprogram的命令式编程。由于大多数命令式语言都具有过程式的特征，所以这两个概念实际具有密切的联系，某些场合可以不加区分。

### 结构化编程

结构化编程（Structured Programming）是一种编程原则，是在过程式基础上发展起来的。

> 结构化定理：任何程序都可以用**顺序（concatenation）**、**选择（selection）**和**循环（repetition）**三种**基本控制结构**来表示。

结构化定理具有非常重要的意义。稍微具备计算机编程基础的人对此都不会感到陌生。

结构化编程还有一个SESE原则，即Single Entry, Single Exit. 顺序、选择、循环每个基本结构都要满足这个单入口、单出口的原则。（这个原则实际上是模拟了电路设计。）

结构化编程的总结：

> 微观上要求循规蹈矩，采用顺序、选择和循环；宏观上要求自顶向下设计，通过模块化将复杂系统分解为多个子系统，最终分解为三种基本结构。

## 声明式编程

声明式编程和命令式编程是相对的。

声明式编程由若干规范（specification）的声明组成，即一系列陈述句，强调**做什么**，而非**怎么做**。

声明式编程起源于**人工智能**的研究。现在人工智能成为热点话题，实际上人工智能的研究早在几十年前就起步了。

声明式编程主要包括**函数式编程和逻辑式编程**。

函数式编程和逻辑式编程都具有悠久的历史，但以前多用于学术研究而非商业应用。

### 逻辑式编程

函数式编程在后文会单独讲解。在此之前先了解下逻辑式编程。

逻辑式编程的代表性语言是Prolog. Prolog，是**Pro**gramming in **Lo**gic的缩写，是一种以一阶谓词为基础的逻辑性语言，是人工智能通用程序设计语言。

Prolog的三大核心：

- 以一阶谓词逻辑为基础的Horn子句集为语法
- 以Robinson的消解原理为工具
- 深度优先的控制策略

关于逻辑式、Prolog和人工智能，涉及到很深刻的理论知识，需要深厚的科学和数学背景才能理解。这里只是简单列一下概念，当做科普。

## 声明式编程和命令式编程的比较

区别：

- 命令式编程是**行动导向（Action-Oriented）**的， 声明式编程是**目标驱动（Goal-Driven）**的
- 命令式算法是显性的，目标是隐性的；声明式算法是隐性的，目标是显性的
- 命令式编程模拟电脑，声明式编程模拟人脑

联系：

- 从编程语言上看，这两种范式相互影响、相互渗透；近年来偏向命令式的语言逐渐开始支持声明特性

BTW，广义的声明式语言其实很常见，如SQL，HTML，SVG，CSS，但他们并非图灵完备，不具有通用性。然而从工程角度看图灵完备与否，对工程本身意义不大。

下面通过一个简单的 Python 的例子，演示命令式和声明式的特点。

例：

在1，2，3，4中取任意三个数字，列出所有排列。

使用REPL:

```python
>>> x = range(1, 5)
>>> result = []
>>> for i in x:
        for j in x:
            for k in x:
                if i != j and j != k and i != k:
                    result.append((i, j, k))
>>> result
```
这段代码具有鲜明的命令式风格：使用事先声明的result保存结果，使用嵌套for循环精确控制细节。

使用声明式风格如何实现？只需一句话即可：

```python
>>> x = range(1, 5)
>>> result = [(a, b, c) for a in x for b in x for c in x if i != j and j != k and i != k]
>>> result
```


BTW， 实际就这个例子本身而言，完全可以采用已有的函数，因为这是一个数学问题。

```python
>>> x = range(1, 5)
>>> from itertools import permutations
>>> p = permutations(x, 3)
>>> result = list(p)
>>> result
```

## 三大核心编程范式

至此，我们已经了解了命令式、函数式和逻辑式，这三种编程范式被称为三大核心编程范式。

总的来讲，编程范式除了命令式就是声明式。

<img src="/fp/prog-paradigm-categories.png">


三大核心编程范式的比较：

|范式|程序|输入|输出|程序设计|程序运行|
|:---:|:---:|:---:|:---:|:---:|:---:|
|命令式|自动状态机|初始状态|最终状态|设计指令|命令执行|
|函数式|数学函数|自变量|因变量|设计函数|表达式转换|
|逻辑式|逻辑证明|题设|结论|设计命题|逻辑推理|


## 面向对象

### OOP与其他编程范式
 
不知识是否留意到，上面的图中 OOP 哪里去了？

答案是 OOP 和它们不在一个范畴，简单讲，没什么关系。严格讲，是**正交**的关系。（数学上把两个相互垂直的向量称作正交。在计算机领域，两个组件如果互相没有什么影响，就可以称作是正交的。）

纯粹的OOP是不存在的，必须结合其他范式而存在。OOP虽然是从命令式发展而来，但这只是历史事实（可能带有某种历史上的必然性），但OOP的思想并不是跟命令式绑定的，也可以应用到函数式编程。

支持OOP的语言大多是命令式，但也存在函数式和逻辑式的OO语言。

**OOP与三大核心编程范式正交，并且越来越广泛地向它们渗透。**

### 面向 vs 导向

**Oriented**意思是“以...为方向，以...为目的”。由于着眼点和思维方式不同，不同编程范式各有侧重，一些编程范式使用oriented来表示一种倾向。

在大陆，Object-Oriented一般译作“面向对象”。这其实是有问题的。“面向”的宾语往往是已经确定的目标，“面向对象”使人误以为对象已经存在，这种译法让人莫名其妙，不知所谓。

在港台地区，Object-Oriented译作**“物件导向”**，且不说“物件”恰当与否，“导向”实际上比“面向”高很多。又如，经济学中“Market-Oriented”译作“市场导向”，而不是“面向市场”。

总之，“对象导向”更能反映这个范式本身的精髓——在设计中以对象为导向。

### OOP的优点和局限性

OOP的思想无需多说。稍微有点计算机编程基础的人都不会陌生。它更接近人的思维方式，将函数按主体执行者分组，并与数据封装到一起，易于理解和使用。在传参方式上变得更加简洁。在面向对象流行之前，只有函数和参数；有了面向对象，原先作为参数的执行者在语法形式上可以放到点操作符的前面，如`method(obj, arg)`变为`obj.method(arg)`.

OOP是有局限性的。当初之所以流行是有商业利益推动。OOP有一段时间极其流行，近些年稍微冷却下来，人们对OOP的看法也趋于理性。归结起来，OOP流行的原因如下：

- 人们误以为OOP容易学习（实际在实践中很难把握其精髓）
- 人们以为OOP会使代码更容易复用（这是个错觉）
- OOP被一些商业公司过分宣传
- OOP创造了一个新兴软件产业

关于OOP的局限性，网上有不少资料。这里仅列举两个牛人说过的话：

{{< blockquote author="John Carmack（第一人称射击游戏之父）" >}} 有时，优雅的实现只需要一个函数。不是一个方法，不是一个类，不是一个框架，只是一个函数。 {{< /blockquote >}}


{{< blockquote author="Joe Armstrong（Erlang创始人）" >}} 面向对象编程语言的问题在于，它总是附带着所有它需要的隐含环境。你想要一个香蕉，但得到的却是一个大猩猩拿着香蕉，而且还有整个丛林。 {{< /blockquote >}}

## 函数式思想

函数式编程可以看做声明式编程的一个子集。近年来，命令式编程语言越来越多地引入函数式（声明式）编程特性。函数式编程已然成为主流编程语言的标配。

实际上很多人已经在不同程度上使用函数式编程了，如很多场合常见的map, filter，reduce,. 其中体现的是与命令式截然不同的思想：不使用选择和循环（看不到if和for），而选择和循环是结构化编程的两个核心。

不同语言的函数式能力可能有很大差别。一门语言是不是函数式语言，其标准是有争议的，往往会陷入无休止的争论。我们可以简单理解为一种程度上的差别。重要的是要领会和理解函数式思维方式。

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
