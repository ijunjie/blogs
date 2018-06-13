---
title: "浅谈设计模式"
date: 2018-05-19
draft: false
tags:
- design patterns
categories:
- arch
---


推荐一个极好的设计模式学习网站 [http://java-design-patterns.com/](http://java-design-patterns.com/). 截至 2018 年5 月份，这个 repo 在 github 的 stars 已超过 10,900+. 这个 repo 涉及的设计模式已经远远超出了 GOF 的 23 种设计模式。当然，本来设计模式就远不止这 23 种。

设计模式的价值在于其思想，而不是模式本身。应该重点从目的而不是结构来认识和学习设计模式。很多设计模式并不是定式，实际使用中有很多变化。

GOF 的设计模式已经出版二十多年了，**有的模式早已内化为语言特性**，因此在今天看来有的模式仿佛有过度设计之嫌，因为这些年来编程语言和软件工程处于不断发展变化中。

**设计模式的道理千条万絮，归根结底就是一句话，抽象的艺术。**


本篇对 23 种设计模式中常用的一些做粗浅的总结。其他一些模式，后续可能会补充到本文，也可能再也不补充，看心情。


## 分类

GoF 将设计模式可以分为三类：

- 创建型：Factory Method、Abstract Factory、Singleton、Builder、Prototype
- 结构型：Adapter、Decorator、Proxy、Facade、Bridge、Composite、Flyweight
- 行为型：Strategy、Template Method、Observer、Iterator、Chain of Responsbility、Command、Memento、State、Visitor、Mediator、Interpreter

还可以按操作类还是对象来分类，大多数模式是针对对象的。这意味着基于对象的语言，如 JavaScript 也可以应用大部分设计模式。

这些分类实际没多少意义，仅仅是一种理解和抽象。


## 创建型 5 种

- Factory Method
- Abstract Factory
- Singleton
- Builder
- Prototype(略)

创建型模式都在想方设法避免 new 的使用。new 本身的操作是极快的，但对象初始化过程可能极慢。总的来说，Java 代码里直接用 new 显得很低级。

工厂模式、单例模式非常重要，常与反射结合使用，与 IoC 和 DI 有千丝万缕的联系；Builder 模式也是一种重要的设计技巧，用来化解对象构建过程复杂性；原型模式在 Java 中不是特别重要。

### Factory Method & Abstract Factory

工厂方法和抽象工厂这两种模式其实是一回事，只是复杂程度不同。

简单工厂模式是工厂方法的一个特例，很难称得上是一种设计模式。简单工厂将实例对象的创建逻辑集中到一起，扩展不方便。如果实例类型过多，简单工厂的代码会很臃肿。

工厂方法模式将对象创建推迟到子类实现。看似是推迟，实际并不是推迟，因为**抽象类的本质意义是代码复用**

```java
public abstract class PizzaStore {
 
	abstract Pizza createPizza(String item);
 
	public Pizza orderPizza(String type) {
		Pizza pizza = createPizza(type);
		System.out.println("--- Making a " + pizza.getName() + " ---");
		pizza.prepare();
		pizza.bake();
		pizza.cut();
		pizza.box();
		return pizza;
	}
}
```

例如：

|产品系列|小杯|中杯|大杯|
|:---|:---|:---|:---|
|冰啤酒|小冰|中冰|大冰|
|常温啤酒|小常|中常|大常|

工厂总是按规格划分。


工厂模式缺点在于结构复杂，难以扩展。如有两种产品系列，三种规格，则有 2x3=6 种产品，三个工厂按各自规格分别完成两个产品系列的生产。此时增加产品系列，则需要修改三个工厂实现；如果增加规格，则需要新增一个工厂实现。

工厂模式往往和反射一起使用，实现 IoC. 


简单工厂到工厂方法的跨越体现了依赖倒置的原则，将 **if 判断转化为类型多态，依赖具体转变为依赖抽象**。工厂模式是实现依赖倒置的最有威力的方式。

抽象，抽象，还是抽象，抽象是计算机世界连接物理世界的桥梁。

### Singleton

单例模式要考虑 jvm 虚拟机、类加载器、反射和序列化的影响。

应尽量将单例的实现委托成熟的平台和框架处理，如 spring，仅在必要时自己实现。

实现时，不建议使用丑陋的 DCl，也不建议使用所谓“饱汉”、“饿汉”式——听名字就使人莫名不爽，不知是哪些无聊的人命名的——建议用静态内部类的方式。

也不建议用 enum, 因为这种方式有点非主流，不为人民大众所广泛接受。也只有 Java 语言的设计者才一厢情愿希望人们将 enum 用于单例，呵呵。

BTW，非主流的东西早晚会被人民扔进历史的垃圾堆，比如 HOCON, 比如 Play Framework，比如 sbt，比如 Scala 的一些奇葩特性。Scala 本来可以成为一门通用且优秀的语言，但被设计者们搞得越来越复杂，设计者的学生要完成论文，拿 Scala 当实验材料。

### Builder

更常见的 Builder 模式为创建一个对象的模式。不需要抽象的 Builder. 目的是在真正构造出对象前，把复杂构建逻辑转移到 builder 对象并做一些集中校验。该模式可以强制实行一种分步骤进行的建造过程。

 
## 结构型 7 种

- Adapter
- Decorator
- Proxy
- Facade
- Bridge(略)
- Composition
- Flyweight(略)


### Adapter

适配器本质上是一个中间虚拟层。不知哪位前辈说过，几乎所有计算机领域的问题都可以通过一个中间虚拟层解决。

用 Head First 设计模式的话来讲，如果一个东西走起路来像鸭子，叫起来像鸭子，那么它就完全可以被当做一只鸭子；然而实际上它有可能是一只安装了适配器的火鸡。组合一个火鸡对象，并实现 Duck 接口，这就是是对象适配器模式。

而类适配器可以继承一个类，同时实现 interface 的其他方法。

还有一种叫做默认适配器，用于减少代码。如 interface 中声明方法过多，如果直接实现 interface 则需要实现所有方法。这时，如果有一个默认的抽象类提供了默认实现，则实现类只需要有选择地实现方法即可。

适配器模式使用过多会让系统非常凌乱。

### Decorator

使用装饰模式会产生比使用继承关系更多的对象。更多的对象会使得查错变得困难，特别是这些对象看上去都很相像。

纯粹的装饰模式在真实的系统中很难找到。一般所遇到的，都是这种半透明的装饰模式。

半透明的装饰模式是介于装饰模式和适配器模式之间的。适配器模式的用意是改变所考虑的类的接口，也可以通过改写一个或几个方法，或增加新的方法来增强或改变所考虑的类的功能。大多数的装饰模式实际上是半透明的装饰模式，这样的装饰模式也称做半装饰、半适配器模式。客户端必须使用这个方法，就必须进行向下类型转换。

**java.io 设计的最糟糕的一点是，没有从命名上区分装饰者和被装饰者，给无数程序员带来了困惑。**


抽象方法可以覆写非抽象方法：

```java
class A {
    void foo() {
        System.out.println("foo");
    }
}

abstract class XA extends A {
    @Override
    abstract void foo();
}
```


### Proxy

静态代理是用委托实现的，代理的也可以用子类继承实现。如果需要很多种代理，则可以看作装饰模式了。

代理模式除了用于 AOP，也常用于延迟初始化。

远程代理表示另一个 JVM 的对象代表；虚拟代理作为创建开销大的对象代表，实现延迟初始化。


## Facade

门面模式可以简化一群类的接口。

门面模式类似于总成车间。组合其他业务对象完成更高层次的功能并暴露给客户端。

门面设模式体现了内外有别的思想。可以对系统接口做层次划分。某种意义上将，Facade 模式是对 public/protected/private 的补充。可以选择性的暴露方法供外部使用，既便于客户端使用，又保证安全性。


### Composition

对象的组合和个别的对象可以抽象为统一的对象。


## 行为型 11 种

- Strategy
- Template Method
- Observer
- Iteratee
- Command
- State
- 其他略

### Strategy

策略模式体现了接口与实现分离的思想和委托思想。

### Template Method

模板方法模式在一个方法中定义一个算法的框架，而将一些步骤迁移到子类中。模板方法使得子类可以在不改变算法结构的情况下，重新定义算法中的某些步骤。

钩子是一种被声明在抽象类中的方法，但只有空的或默认的实现。子类可以选择覆盖也可以选择不覆盖。

工厂方法可以看作一种特殊的模板方法。

策略模式用组合的方式封装算法，模板方法用继承的方式封装算法。

模板方法的优点同时也是它的缺点，子类只知道覆写钩子方法，不知道上层如何调度。就像绣春刀II里的人物不过是高层的棋子而已。

### Observer

Subject 的实现类中包含 Observer 引用; Observer 的实现类中包含 Subject 引用。在 JDK 中有 Observable 和 Observer 实现。

建议不要使用 JDK 自带的，而是自己实现。因为 JDK 的 Observable 和 Observer 是类，不易扩展。

```
interface Observer {
    void update(float temp, float humidity, float pressure);
}

interface Subject {
    void registerObserver(Observer observer);

    void removeObserver(Observer observer);

    void notifyObservers();
}
```

### Iteratee

迭代器模式，提供一个方式遍历集合，无需暴露集合的实现。

### Command

Java 8 函数式新特性的出现，给了 Command 模式巨大冲击。这就是设计模式内化为语言级特性的一个典型例子。一个 Command 对象可以用一个方法引用优雅地实现。但 Java 8 的 Functional 接口只能有一个方法声明。定义 Command interface 更加灵活一些。如在 Command 中声明 execute 和 undo 两个方法。

命令模式可以实现操作撤销。

### State

把状态抽象为独立的类，将动作委托到对象。