---
title: "正确理解 ArrayList 和 LinkedList"
date: 2018-05-02
draft: false
tags:
- fp
categories:
- programming
---


关于 ArrayList 和 LinkedList 的比较，网上流传着很多不准确的说法。比如 “ArrayList 查询快，LinkedList 增删快”，这种说法过于笼统，不严谨，实际情况要复杂一些。下面从它们底层的数据结构和工作机制入手，按场景对比分析它们的性能差异。

影响 ArrayList 性能的关键点在于**数组拷贝**，动态扩容和随机增删元素都会进行数组拷贝。数组拷贝既消耗时间，又消耗空间，应尽量避免之。由数组拷贝机制可以推理得到 ArrayList 的一个特点，**增删元素的位置越往后越快**，因为元素位置越靠后，数组拷贝的规模越小。

影响 LinkedList 性能的关键点在于**寻址**和**内存分配**。LinkedList 寻址慢在于每 get 一个元素都需要**查找**， 即使有折半判断 + 双向查找优化，仍比不上 ArrayList 按“索引”取址. 因此**对 LinkedList 来说，增删的元素位置越靠近两端，效率越高**。 内存分配是说，向 LinkedList 增加或插入元素，每次都需要申请内存。这一点显然不如 ArrayList 的静态分配效率高（不考虑动态扩容）。

## 遍历

总体来讲，ArrayList 的效率比 LinkedList 稍高一些。

值得注意的是，要正确选择 for 循环和 Iterator：

- ArrayList 使用普通 for 循环遍历效率比较高，不要用 Iterator
- 由于 LinkedList 查找元素时寻址性能差，所以不要使用普通的 for 循环遍历 LinkedList, 应该使用 Iterator.
- 关于 foreach， java 不同版本的编译器可能会对 foreach 做一些优化，需要根据实际测试结果判断


## 尾部增加元素

总体来讲，选择 ArrayList 比较好，但要注意，应尽量初始化 capacity，以避免发生动态扩容。

- ArrayList 基于数组，寻址很快，在末尾位置赋值即可；
- LinkedList 虽然寻址较慢，但在末尾插入元素，JDK做了优化，会从末尾查找，所以末尾插入元素，两者在寻址方面实际上效率差别不大。主要差别在于，LinkedList 插入时要新建节点对象，向 JVM 申请内存空间。

有些测试结果可能会显示，末尾增加元素时，LinkedList 性能优于 ArrayList. 这一点跟 ArrayList 动态扩容和集合内元素数量有关系。总的来讲，不较真的情况下，选用 ArrayList 是 OK 的。

## 尾部删除元素

如果总是从尾部删除元素，理论上 ArrayList 和 LinkedList 差别不大。ArrayList 将末尾位置指定为 null，等待垃圾回收，无需数组拷贝；LinkedList 使用优化的寻址方法从末尾查找，对元素执行 unlink. 总体来讲，建议还是使用 ArrayList.

## 增删靠前位置的元素

根据上面的分析，增删元素的位置越靠前，ArrayList 拷贝数组的元素越多，而 LinkedList 寻址的消耗越小，因此在这个场景 LinkedList 具有无可争议的优势。

## 性能测试

ArrayList 和 LinkedList 的性能之比较有很多影响因素，不能简单一概而论，与 ArrayList 动态扩容、集合元素数量、内存分配状况、垃圾回收等都有关系。最稳妥的办法是进行性能测试。