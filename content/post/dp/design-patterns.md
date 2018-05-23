---
title: "设计模式概览"
date: 2018-05-19
draft: true
tags:
- docker
categories:
- arch
---

# 设计模式概览

**23 种设计模式中的价值在于思想，而不是模式本身。有的模式早已内化为语言特性；有的模式几乎是为了设计而设计，并没有多少真正的实用价值。**


设计模式可以分为三类：

- 创建型：Factory Method、Abstract Factory、Singleton、Builder、Prototype
- 结构型：Adapter、Decorator、Proxy、Facade、Bridge、Composition、Flyweight
- 行为型：Strategy、Template、Observer、Iteratee、责任链、命令、备忘录、状态、访问者、中介者、解释器

https://blog.csdn.net/jason0539/article/details/44956775
http://www.cnblogs.com/java-my-life/archive/2012/04.html


## 创建型 5 种

- Factory Method
- Abstract Factory
- Singleton
- Builder
- Prototype

这些模式都在想方设法避免 new 而已。不管有什么实际意义上的理由，Java 代码里直接用 new 总是显得很低级。有时候人类的逻辑很奇怪，发明出的东西又想方设法避免使用。

其中，工厂模式、单例模式非常重要，常与反射结合使用；Builder 模式也是一种重要的设计技巧。

### Factory Method & Abstract Factory

工厂方法和抽象工厂这两种模式其实是一回事，只是复杂程度不同。简单工厂模式是工厂方法的一个特例，很难称得上是一种设计模式。简单工厂将实例对象的创建逻辑集中到一起，扩展不方便。如果实例类型过多，简单工厂的代码会很臃肿。

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

|产品系列|小杯|中杯|大杯|
|:---:|:---:|:---:|:---:|
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

扯点别的话题，非主流的东西早晚会被人民扔进历史的垃圾堆，比如 HOCON, 比如 Play Framework，比如 sbt，比如 Scala 的一些奇葩特性。Scala 本来可以成为一门通用且优秀的语言，但被设计者们搞得越来越复杂，设计者的学生要完成论文，拿 Scala 当实验材料。

### Builder

更常见的 Builder 模式为创建一个对象的模式。不需要抽象的 Builder. 目的是在真正构造出对象前，把复杂构建逻辑转移到 builder 对象并做一些集中校验。该模式可以强制实行一种分步骤进行的建造过程。

### Prototype

略。
 
## 结构型 7 种

- Adapter
- Decorator
- Proxy
- Facade
- Bridge
- Composition
- Flyweight


### Adapter

适配器本质上是一个中间虚拟层。

类适配器可以继承一个类，同时实现 interface 的其他方法。**对象适配器**可以组合一个已有的实现类，并实现 interface 补充实现其他方法。推荐使用后者。

默认适配器用于减少代码。如 interface 中声明方法过多，如果直接实现 interface 则需要实现所有方法。这时，如果有一个默认的抽象类提供了默认实现，则实现类只需要有选择地实现方法即可。

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


## Facade

门面设模式体现了内外有别的思想。可以对系统接口做层次划分。某种意义上将，Facade 模式是对 public/protected/private 的补充。可以选择性的暴露方法供外部使用，既便于客户端使用，又保证安全性。

### Bridge

### Composition

### Flyweight

## 行为型 11 种

- Strategy


### Strategy

策略模式体现了接口与实现分离的思想和委托思想。

### Template

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
