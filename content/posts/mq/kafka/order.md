---
title: "Order"
date: 2024-02-10T22:31:39+08:00
draft: true
---
# 乱序原因
## 生产者发送消息时乱序
生产者对于一个连接，默认情况下允许发送5个未响应的请求到Kafka服务端，因此，当中间存在丢包、网络阻塞等问题时，可能存在后发送的数据先到达服务器，从而造成数据的乱序
### 解决方案
- 并行度
可以通过设置[`max.in.flight.requests.per.connection`](https://kafka.apache.org/documentation/#producerconfigs_max.in.flight.requests.per.connection)为1进行解决，即对于一个连接最多允许同时存在一个未响应的请求。
- 幂等
通过启用幂等，同时控制最大并发请求数不超过5，即可实现生产者单分区的有序性。这是由于服务端在幂等时单连接会缓存最多五个请求的元数据，接受后会通过序列号（递增）排序，有序递增部分则落盘。
## 消费者消费时乱序

