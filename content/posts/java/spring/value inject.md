---
title: "List Value Inject"
# 当前时间
date: 2024-06-22T11:35:00+08:00
# draft: true
series: ["spring"]
weight: 1
draft: true
tags: ["bug"]
author: ["夏旭晨"]
---
# 问题描述
在使用spring的过程中，需要对属性进行注入，但是注入yaml中的列表类型发生报错，
> Could not resolve placeholder 'spring.kafka.producer.bootstrap-servers' in value "${spring.kafka.producer.bootstrap-servers}"

代码如下:
application.yaml
```yaml
bootstrap-servers: localhost:9092
kf: hello,world
spring:
  kafka:
    producer:
      bootstrap-servers:
        - localhost:9092
```
ProducerHelper.java
```java
@Component
@Slf4j
public class ProducerHelper {

    @Value("${bootstrap-servers}")
    private String bootstrapServers;

    @Value("${kf}")
    private List<String> bs;

    @Value("${spring.kafka.producer.bootstrap-servers}")
    private List<String> kafkaServers;
}
```
前面两个都能注入，但是第三个怎么也注入不了
# 进一步探究