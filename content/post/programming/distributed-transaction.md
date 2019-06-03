---
date: 2018-06-03
title: "分布式事务 memo"
draft: false
tags:
- arch
categories:
- programming
comment: true
---






- 分布式理论
	- CAP
	- BASE
- 分布式事务
	- 2PC (两阶段提交) 和 XA 协议
	- TCC（补偿事务）
	- 本地消息表（异步确保）。一种非常经典的实现，避免了分布式事务，实现了最终一致性
	- MQ 事务消息。RabbitMQ 和 Kafka 都不支持，RocketMQ 支持。
	- Sagas 事务模型。鲜为人知。


- [https://www.cnblogs.com/savorboard/p/distributed-system-transaction-consistency.html](https://www.cnblogs.com/savorboard/p/distributed-system-transaction-consistency.html)
- [https://blog.csdn.net/mine_song/article/details/64118963](https://blog.csdn.net/mine_song/article/details/64118963) 