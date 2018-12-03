
---
date: 2018-06-03
title: "聊聊分布式事务"
draft: false
tags:
- arch
categories:
- programming
comment: true
---


丢个链接闪人：[https://www.cnblogs.com/savorboard/p/distributed-system-transaction-consistency.html](https://www.cnblogs.com/savorboard/p/distributed-system-transaction-consistency.html)

算了，还是写点导读。上面链接是关于分布式事务极其解决方案写的比较好的一篇文章。主要总结了如下问题：

- 分布式理论
	- CAP
	- BASE
- 分布式事务
	- 2PC (两阶段提交) 和 XA 协议
	- TCC（补偿事务）
	- 本地消息表（异步确保）。一种非常经典的实现，避免了分布式事务，实现了最终一致性
	- MQ 事务消息。RabbitMQ 和 Kafka 都不支持，RocketMQ 支持。
	- Sagas 事务模型。鲜为人知。


再丢个链接 [https://blog.csdn.net/mine_song/article/details/64118963](https://blog.csdn.net/mine_song/article/details/64118963) ，闪人。