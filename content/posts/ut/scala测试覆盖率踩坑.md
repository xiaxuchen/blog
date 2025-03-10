---
title: "Scala测试覆盖率踩坑"
date: 2025-02-27T18:30:14+08:00
draft: true
---

# Scala测试覆盖率踩坑

## 一、 场景

```scala
if (childStatus == GlutenCacheStatus.PARTIAL_HIT
        || childStatus == GlutenCacheStatus.MISS && result == GlutenCacheStatus.HIT
        || childStatus == GlutenCacheStatus.HIT && result == GlutenCacheStatus.PARTIAL_HIT)
```

![alt text](/images/2025-02-27-17-24-09-image.png)
![alt text](/images/2025-02-27-17-23-15-image.png)
![alt text](/images/2025-02-27-17-22-44-image.png)
可以看到，三行需要覆盖的分支都很奇怪，非常多，但实际上并不可能有这么多的branch



## 二、 问题排查

```java
 L1
    LINENUMBER 114 L1
    ALOAD 3
    GETSTATIC GlutenCacheStatus.PARTIAL_HIT : LGlutenCacheStatus;
    ASTORE 4
    DUP
    IFNONNULL L2
    POP
    ALOAD 4
    IFNULL L3
    GOTO L4
```

以上是`if (childStatus == GlutenCacheStatus.PARTIAL_HIT)`的字节码，在其中可以看到额外的分支，**包括IFNONNULL和IFNULL，这些都是scala编译出来的**，而我的测试并没有覆盖(主要原因是代码逻辑上枚举并不会赋值null，且无法通过函数的输入使得枚举值为null)，因此导致此问题

## 三、 问题解决

**使用eq替代==，scala不会再编译出IFNULL和IFNONNULL**

```java
L1
    LINENUMBER 114 L1
    ALOAD 3
    GETSTATIC GlutenCacheStatus.PARTIAL_HIT : LGlutenCacheStatus;
    IF_ACMPEQ L2
```

## 四、 总结

近期对于scala的使用，明显感觉到了scala的许多问题，虽然scala在一些语法糖方面给到了便利，但是其也给代码带来了许多隐含的含义，对于不熟悉的Javaer不是很友好。
