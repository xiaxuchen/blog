---
title: "Kafka:Partition"
date: 2024-02-09T13:56:36+08:00
draft: true
---

# 指定分区发送

Kafka 生产者 Api 提供的ProducerRecord 对象，能够直接指定分区对消息进行发送。

# 分区策略

但是正常情况下，对于消息的生产者，正常情况下是**不感知分区**的。Kafka 默认的分区策略为:"根据 Key 的 Hash 值对总分区数取模，而如果没指定 key 则使用 Sticky 分区策略，即**分区待发送队列**随机选定一个分区后就一直往该分区发送直到满".Kafka 也提供分区策略的自定义，我们可以根据自己的需求进行分区选择，如根据分区的距离就近发送，根据分区的容量均衡发送。

## 自定义分区器

```
public class MyPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> infos = cluster.availablePartitionsForTopic(topic);
        // 实现基于key默认的分区策略
        return key.hashCode() % infos.size();
    }

    @Override
    public void close() {

    }

    @Override
    public void configure(Map<String, ?> configs) {

    }
}

// 配置到producer的配置中
props.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, MyPartitioner.class.getName());
```

# 分区副本重新分配
上线新节点或者下线旧节点时，需要对partition的数据进行迁移。对于新节点我们需要将数据迁移到新节点上，而对于旧节点则需要从旧节点迁移走数据，这样我们可以通过json指定待迁移的topic名称，随后通过kafka自带的命令生成迁移计划，如下:
```
kafka-reassign-partitions.sh --bootstrap-server broker地址 --topics-to-move-json-file topics-to-move.json --broker-list "0,1" --generate
```
其中"0,1"可以根据需求改成最终存储数据的brokerid集合即可

通过该命令会生成执行计划，我们可以将执行计划写入一个文件，这里我们叫做increase-replication-factor.json,随后输入以下命令即可执行计划

```
bin/kafka-reassign-partitionss.sh --bootstrap-server broker地址 
--reassignment-json-file increase-replication-factor.json --execute
```

通过将--execute改成--verify，即可在执行后进行验证是否与执行计划相同
