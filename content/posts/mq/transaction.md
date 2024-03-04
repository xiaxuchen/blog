---
title: "消息队列事务"
date: 2024-02-10T20:35:11+08:00
summary: "消息队列事务如何保证事务的一致性？"
series: ["消息队列"]
weight: 5
tags: ["消息队列"]
author: ["夏旭晨"]
---

# Kafka 如何保证事务的一致性？

## 数据及状态存储

Kafka 实现事务消息是基于两阶段提交进行的，消息本身仍然发送到对应的 topic 的分区中，而事务的状态是存储在\_\_transaction_state topic 中，由事务的协调者(TransactionCoordinator)进行控制。

## 可见性控制

每个分区都有 LSO(log stable offset)标志位对消息的可见性进行控制，对于一个分区，LSO 指定位置及后续的消息都对 readCommitted 隔离级别的消费者(默认[isolation.level](https://kafka.apache.org/documentation/#consumerconfigs_isolation.level)是 read_uncommitted)不可见.而当事务提交后，事务的协调者会通知对应的 broker 往后移动 LSO 使得事务消息可见。而若事务回滚，也会通知 broker 记录事务相关的元信息(producerId, 事务第一条消息位点 startOffset 及事务最后一条消息的位点 endOffset)进入`.txnindex`文件中，使得消费者在读取消息的时候排除掉回滚的消息。分区维度的`.snapshot`文件会记录消息事务的生产者的 producerId。消费者拉取消息时会过滤掉 startOffset 和 endOffset 之间，指定 producerId 的消息。

## 事务提交

当消息都发送到对应的 topic 之后，生产者通过调用 commit 请求事务协调者将事务状态变更为 PrepareCommit,在这时该事务已执行完成，而消息的可见性则会由事务的协调者异步的发送给 broker 进行 LSO 位点的移动。

## 事务回滚

当事务回滚的时候，事务的协调者会将状态设置为 PrepareAbort,同时通知事务参与者写`.txnindex`文件记录事务的位点区间。

## 最终一致性

上述提交和回滚过程可以看出，Kafka 的事务并不能实现强一致性，而是通过异步使得事务中的消息实现消息可见性的最终一致性。

# 参考

[消息队列中的事务](https://blog-git-preview-sspirits.vercel.app/p/what-is-trasaction-in-message-queue)
