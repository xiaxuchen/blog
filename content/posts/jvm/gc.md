---
title: "Gc"
date: 2024-03-13T21:45:09+08:00
draft: true
---

# 一、垃圾收集器
## 分类
### 根据回收区域划分
- 年轻代
- 老年代
- 整个堆
### 串行、并行还是并发
- 串行Stop the world
- 并行多线程Stop the world
- 部分Stop the world，部分与用户线程并发执行
### 收集算法
- 复制算法(年轻代使用)
- 标记清除(CMS)
- 标记整理(老年代使用)
- 划分Region(以Region为视角是复制算法，从一个Region复制到另一个Region,整个堆来看是标记整理算法)
# 二、 GC分类
## Minor GC
年轻代GC,当**年轻代满**了，就会进行回收，**需要STW**,会从survivor from区和Eden区复制对象到Survivor区，如果分代年龄到了阈值或者survivor to区内存不够，那么就会晋升老年代.

### 注
- 年轻代:分为Eden和Survivor from和Survivor to区，默认是1:1:8的比例，每次Young GC的时候会从Eden和from区将存活的对象复制到to区。
- STW: stop the world，暂停用户线程活动

## Major GC（Old GC）
老年代GC，只有CMS有这一行为，专门回收老年代的堆内存。而Major GC并非老年代满了才进行，而是根据设置的阈值启动。

## Mixed GC
混合GC。G1收集器特有，混合回收年轻代和部分老年代，可以根据需要设置阈值进行Mixed GC回收。通过Mixed GC可以提早多次分批回收内存。

## Full GC
整堆GC。需要避免。一般都是降级为Serial进行GC.
### 触发Full GC的条件
- System.gc()
- 老年代空间不足
- 空间分配担保失败
- 使用CMS（Concurrent Mark-Sweep）垃圾回收器时的并发失败(并发收集时老年代空间不足)
- **永久代满**
### 空间分配担保机制
即当Young GC时，判定当前老年代是否有连续空间存储年轻代的对象，如果有就Young GC.而若没有，若启用空间分配担保配置，则判定当前老年代连续空间剩余是否大于Young GC历次平均晋升老年代的对象大小,若是则Young GC,否则Full GC

### Full GC排查
- 开启Dump当Full GC的时候
-XX:HeapDumpBeforeFullGC
- 根据Dump文件查看对象占比，排查可疑对象
- 查看可疑对象的引用关系

# 三、 收集器
## Serial收集器
最早的收集器，单线程收集，年轻代，复制算法，只能与Serial Old和CMS配置
## Serial Old
老年代版本，标记压缩算法
## ParNew
Serial年轻代多线程版本，复制算法，只能与Serial Old和CMS配置
## Parallel
多线程吞吐量优先年轻代，复制算法，只能与Parallel Old和Serial Old配合
## Parallel Old
多线程吞吐量优先老年代,标记压缩算法
## CMS
并发标记清除收集器，标记清除算法，与用户线程并发，低延迟
### Concurrent Mode Failure
执行 CMS GC 的过程中同时有对象要放入老年代，而此时老年代空间不足(可能是 GC 过程中浮动垃圾过多导致暂时性的空间不足)，便会报 Concurrent Mode Failure 错误，并触发 Full GC。
## G1
分Region的收集器，将对划分为多个规整的Region，基于Region进行老年代，年轻代，大对象区域的划分。

# 四、 永久代和元空间的差别
