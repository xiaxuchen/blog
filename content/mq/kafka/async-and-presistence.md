---
title: "Kafka:Async and Persistence"
date: 2024-02-09T13:09:10+08:00
draft: true
series: ["深入学习Kafka"]
weight: 5
tags: ["消息队列", "Kafka"]
author: ["夏旭晨"]
---

# 同步和异步

在使用 Kafka 的时候，我们通常讨论的同步和异步是与我们业务逻辑之间的，即我们的业务是否需要等待消息发送到 Kafka，但是**同步并不保证消息的落盘**，这与后面我们要讲的**应答机制**有关。而 Kafka 中是否进行同步发送在于使用方是否等待 Future 返回，若调用 send 但不理会返回值，则为异步(RocketMQ 称为 oneWay，即不关心是否发送成功，消息发送到了哪)，而如果调用`producer.send().get()`即为同步等待消息发送，获取到消息的响应信息。而我们如果需要异步发送，但是仍关心消息在 broker 中的元信息，我们可以在 send 方法中添加 Callback 回调函数来异步接收请求的响应。如下:

```
Future<RecordMetadata> future = producer.send(new ProducerRecord<>("bar", "key", "value"));
// 同步发送需额外调用get方法等待响应
future.get();
// 若需要异步，且关心响应信息
producer.send(new ProducerRecord<>("bar", "key", "value"), (metadata, exception) -> {
    if (exception != null) {
        // 发送失败，获取异常信息
    } else {
        // 发送成功，获取元数据信息
    }
});
```

# 应答机制(acks)

Kafka 可以配置参数[acks](https://kafka.apache.org/documentation/#producerconfigs_acks)来控制是同步落盘还是异步落盘。

- 默认情况下，该配置为 0，即无需任何节点落盘即返回响应。

  这将可能导致消息丢失，产生数据不一致。比如当我们发送消息后响应了，我们的业务认为消息发送成功，但是返回响应后节点宕机了，那么消息就丢失了。

- 配置为 1 切换为 leader 节点落盘就响应
- 配置为-1 使得所有 isr 都落盘成功才进行响应

  这样能够提高数据的一致性，但同时也牺牲了性能。因此我们需要根据业务场景选择具体的配置，比如我们发送日志的时候，我们可以使用 0 来提高发送效率，而对于强一致性的需求时如订单消息我们则可以采用-1 来保证数据的一致性。

  而如果 isr 中的副本节点超过[replica.lag.time.max.ms](https://kafka.apache.org/documentation/#brokerconfigs_replica.lag.time.max.ms)未与 leader 通信将被剔出 isr。默认为 30s。

  当 isr 中只有 leader 节点，那么-1 的效果和 1 是一样的。因此同样无法保证消息发送的可靠性，因此我们还需要配置 topic 的[min.insync.replicas](https://kafka.apache.org/documentation/#topicconfigs_min.insync.replicas)大于 1，才能保证至少有一个副本落盘。
  即完全可靠性保证为，replications > 1 && acks == -1 && min.insync.replicas > 1

# 问题
acks=-1时，leader节点持久化后，宕机了，那么消息是否算是发送成功了，还是说有两阶段提交机制。重新选择leader后，重发消息，持久化后，原leader上线了，那么他需要同步消息吗？