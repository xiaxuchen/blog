---
title: "Collections"
date: 2024-03-02T22:13:21+08:00
draft: true
---

# ArrayList

## 扩容

可以手动调用 ensureCapacity 方法，来直接扩容到指定容量。

如果使用默认的构造方法，那么就会默认赋值空数组，当扩容的时候会尝试采用**默认容量 10**
而扩容的逻辑为：

```
Math.max(oldCapacity + oldCapacity >> 1, minCapacity);
```

## 快速失败

在做修改操作的时候会 modCount++,在迭代器创建的时候会记录当前的 modCount，然后在调用任意迭代器的方法都会触发检查 modCount 是否变化，如果变化就会抛出**ConcurrentModificationException**。

这就导致我们无法在使用迭代器遍历的时候去**写容器**。

# LinkedList(双向链表)

## 接口

实现了 List 接口、Deque 接口和 Queue 接口，因此可以当做线性表、队列、栈去使用，**Stack 已经不建议使用了**
**而如果需要使用队列或者栈，可以使用性能更优的 ArrayDeque**

# LinkedHashMap

## 特点

在 HashMap 的基础上，采用双链表维护了插入的顺序

## FIFO 实现

```
/** 一个固定大小的FIFO替换策略的缓存 */
class FIFOCache<K, V> extends LinkedHashMap<K, V>{
    private final int cacheSize;
    public FIFOCache(int cacheSize){
        this.cacheSize = cacheSize;
    }

    // 当Entry个数超过cacheSize时，删除最老的Entry
    @Override
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
       return size() > cacheSize;
    }
}
```

# Map

## HashMap

### capacity 2^k

将取模运算替换为与运算，优化算 hash 槽

## WeakHashMap

### WeakReference 与 ReferenceQueue

当弱引用被回收时，WeakReference对象会被加入到其绑定的 ReferenceQueue 中(真正的弱引用已经被回收)，用以释放其他相关的对象，如 WeakHashMap的value。
WeakHashMap的弱引用是Key.

而 WeakHashMap 进行任意操作时，都会触发队列的清理操作，防止内存泄漏。
# Queue

## PriorityQueue(小根堆)

### 小根堆的插入和删除
当小根堆插入时，会将元素放到当前数组最后一个，然后向上对比是否比父元素大，去上浮

而删除元素的时候，直接将堆顶删除，然后将最后一个元素放到堆顶，进行比较，将子元素进行上浮。

# Deque

## ArrayDeque(循环队列)

### 指针

head 为头，head 指向头的位置，如果要插入则 head - 1 位置插入，tail 为尾部，tail 指向尾部插入的第一个位置

### 2^k 大小

使用 2 的次方作为大小，便于对 head、tail 等索引求余，保证不超出索引。

主要为了采用与运算代替取模运算。

### 扩容

因为采用 2^k 作为大小，因此扩容是双倍的扩容，当添加一个元素后，head 与 tail 相遇，那么说明满了，就会进行扩容。
