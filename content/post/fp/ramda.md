---
title: "Ramda：一个精致的Javascript函数式编程库"
date: 2018-04-22
draft: false
tags:
- fp
categories:
- fp
---

# Ramda是什么

Ramda的官方文档中文版：[Ramda 简介](https://adispring.coding.me/2017/06/25/Introducing-Ramda/)

阮一峰也曾以他一贯的通俗易懂的风格写过一篇介绍：
[Ramda 函数库参考教程](http://www.ruanyifeng.com/blog/2017/03/ramda.html)

# 柯里化

关于什么是柯里化，可以参考以下文章。当然也可以通过别的FP语言了解柯里化。

- [爱上柯里化](https://adispring.coding.me/2017/06/27/Favoring-Curry/)
- [为什么柯里化有帮助](https://adispring.coding.me/2017/06/28/Why-Curry-Helps/)

正如其他FP语言一样，Ramda将柯里化几乎运用到了极致。其总是将**数据参数放到最后**的设计具有非常重大的意义。数据参数传入之前通过**函数组合**产生Partial Application, 这一点也正是FP的精髓。

# 快速上手

使用jsfiddle快速体验Ramda. jsfiddle是一个在线环境，可以方便的运行html/js/css代码，非常适合用做代码学习和demo演示。

下面的jsfiddle embed代码由于使用了script async异步加载，可能稍后才会显示。

如果还不能显示需自行解决网络问题。

<script async src="//jsfiddle.net/wangjunjie/1rooqpp5/33/embed/js,html,result/"></script>



这里面比较有趣的一点是R.flip函数。R.flip用于交换函数的两个参数。

如R.append的参数列表是(ele)(arr), 则R.flip(R.append)等同于`(acc, value) => acc.concat(value)`

 `R.reduce(R.flip(R.append))([])(data)`

相当于

`R.reduce((acc, value) => acc.concat(value))([])(data)`.