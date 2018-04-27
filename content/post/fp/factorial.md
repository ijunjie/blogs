---
title: "匿名函数的递归调用"
date: 2018-04-27
draft: false
tags:
- fp
- "tail rec"
categories:
- fp
---

一般的递归或者尾递归都要**显式地调用递归函数**，那么匿名函数如何实现递归调用？

这里用Javascript写了一个求阶乘的例子：

```javascript
let factorial = n => (f => n => s => f(f)(n)(s))
(f => n => s => {
  if (n == 1) return s;
  let m = n - 1;
  return f(f)(m)(s * m);
})
(n)
(n);

console.log(factorial(5));
```

在函数式编程方面训练有素的程序员对这段代码不会感到困难。这段代码中用到了高阶函数、柯里化、尾递归、IIFE等技巧。

下面解释如何一步步写出上面这段代码。

## 递归和尾递归

这一节参考了`Takafumi Shido`的`Yet Scheme Another Tutorial`, 中文版[《Scheme入门教程》](http://deathking.github.io/yast-cn/).

首先看一下什么是递归和尾递归。通常使用计算阶乘来解释递归。这里使用Scheme语言：

```scheme
(define (fact n)
  (if (= n 1)
      1
      (* n (fact (- n 1)))))
```

这段代码使用了递归，但不是尾递归。`(fact 5)`的计算过程如下：

```
(fact 5)
⇒ 5 * (fact 4)
⇒ 5 * 4 * (fact 3)
⇒ 5 * 4 * 3 * (fact 2)
⇒ 5 * 4 * 3 * 2 * (fact 1)
⇒ 5 * 4 * 3 * 2 * 1
⇒ 5 * 4 * 3 * 2
⇒ 5 * 4 * 6
⇒ 5 * 24
⇒ 120
```

`(fact 5)`调用`(fact 4)`，`(fact 4)`调用`(fact 3)`，最后`(fact 1)`被调用。`(fact 5)`，`(fact 4)`……以及`(fact 1)`都被分配了不同的存储空间，直到`(fact (- i 1))`返回一个值之前，`(fact i)`都会保留在内存中，由于存在函数调用的开销，这通常会占用更多地内存空间和计算时间。

更环保的方式是**尾递归**。尾递归函数包含了计算结果，当计算结束时直接将其返回。尾递归在很多语言中采用编译期转化为循环实现。

下面是尾递归版本：

```scheme
(define (fact-tail n)
  (fact-rec n n))

(define (fact-rec n p)
  (if (= n 1)
      p
      (let ((m (- n 1)))
    (fact-rec m (* p m)))))
```

`fact-tail`计算阶乘的过程：

```
(fact-tail 5)
⇒ (fact-rec 5 5)
⇒ (fact-rec 4 20)
⇒ (fact-rec 3 60)
⇒ (fact-rec 2 120)
⇒ (fact-rec 1 120)
⇒ 120
```

因为fact-rec并不等待其它函数的计算结果，因此当它计算结束时即从内存中释放。计算通过修改fact-rec的参数来演进，这基本上等同于循环。

## Javascript实现

现在我们使用Javascript实现上面一节中阶乘的例子， 使用尾递归。

```javascript
let fact_rec = (n, s) => {
  if (n == 1) return s;
  let m = n - 1;
  return fact_rec(m, s * m);
};

let fact = n => fact_rec(n, n);
```

首先定义了一个fact_rec函数，其中显式的递归调用了自身；另一个fact函数中再次调用fact_rec. 那么如何使用一个函数、并且使用匿名函数实现尾递归调用？

首先，改为匿名函数。匿名函数的递归调用是通过参数自调实现的。

定义一个辅助函数：

```javascript
let temp = f => n => s => f(f)(n)(s);
```

temp接收三个参数，第一个参数即真正的阶乘实现，后面两个参数是最终调用的初始参数值。求5的阶乘：

```javascript
temp(f => n => s => {
  if (n == 1) return s;
  let m = n - 1;
  return f(f)(m)(s * m);
})
(5)
(5);
```

然后，将5再次提取成参数，将temp做IIFE调用即完成最后版本：

```javascript
let factorial = n => (f => n => s => f(f)(n)(s))
(f => n => s => {
  if (n == 1) return s;
  let m = n - 1;
  return f(f)(m)(s * m);
})
(n)
(n);

console.log(factorial(5));
```

这里面关键在于**匿名函数通过形式参数实现了递归调用**。匿名函数没有名字，只能将匿名函数作为参数传递实现递归调用。了解更多，请参考我的另一篇博文相关章节的示例： [浅谈函数式编程（高阶函数）](https://ijunjie.github.io/post/fp/fp-basic/#%E7%BB%BC%E5%90%88%E7%A4%BA%E4%BE%8B)