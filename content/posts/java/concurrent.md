---
title: "Concurrent"
date: 2024-03-04T23:30:24+08:00
draft: true
---

# 如果提前调用 Interrupt 会怎么样？

测试代码如下所示，运行后快速就停止了，并打印了 Interrupted

```
public static void main(String[] args) throws InterruptedException {
        long l = System.currentTimeMillis();
        Thread thread = new Thread(() -> {
            while (System.currentTimeMillis() < l + 500) {
                System.out.println("test");
            }
            try {
                Thread.sleep(10000);
            } catch (InterruptedException e) {
                System.out.println("interrupted");
                throw new RuntimeException(e);
            }
        });
        thread.start();
        thread.interrupt();
        thread.join();
    }
```

# 读写锁的共享锁是如何记录的

通过 ThreadLocal<HoldCount> 进行记录，当 release 的时候可以从 ThreadLocal 中取。HoldCounter主要有两个属性，count和tid,count为重入次数，tid为线程id。

## 读写锁的降级
对于tryAcquireShared，是支持当前线程持有写锁的时候设置读锁的，这样也就是最多一个线程会同时获取到读写锁，而后释放掉写锁，即可让其他读锁读取。

## tryLock 超时

tryLock 会尝试 tryAcquire，如果失败则进入队列，然后开始循环，park 时会设置超时时间，如果超时时间到了，那么就会退出循环，并取消掉节点。

```
private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L)
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    // 这里根据超时时间进行park
                    LockSupport.parkNanos(this, nanosTimeout);
                // 能响应Interrupt
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            // 取消节点
            if (failed)
                cancelAcquire(node);
        }
    }
```

## lock

lock 是不能 interrupt 的

## 共享 lock

当是共享 lock 的时候，需要唤醒后继节点，后继节点如果是共享的，那么也能 lock 成功，形成递归。
而如果是独占 lock，那么就会 park。

而 release 的时候，同样会唤醒后继节点。让后继节点进行加锁。

## ConditionObject

是 AQS 的内部类，因此能拿到 AQS 的数据，如当前的头结点以及锁的状态。当 ConditionObject.await 的时候，就会将当前的线程加入 ConditionObject 中，释放当前的所有冲入次数然后等待 Condition 条件唤醒，通过 waiterNext 串起来。
因为只有获取到锁的线程才能操作 Condition，因此**不需要进行同步**。

# ConcurrentHashMap

## jdk1.7

### Segment put 操作

#### 为什么会有 scansAndLockForPut？

为了最大化利用 cpu，加快操作的执行，当获取锁失败，重试去遍历**一步而不是一遍**hash 槽中的数据，判定是否在 hash 槽中存在当前元素，如果不存在，就可以**创建出元素**。那么等到加锁后真正遍历的时候就能够直接头插法进去了。

#### 插入元素

头插法

#### 扩容

```
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 在往该 segment 写入前，需要先获取该 segment 的独占锁
    //    先看主流程，后面还会具体介绍这部分内容
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        // 这个是 segment 内部的数组
        HashEntry<K,V>[] tab = table;
        // 再利用 hash 值，求应该放置的数组下标
        int index = (tab.length - 1) & hash;
        // first 是数组该位置处的链表的表头
        HashEntry<K,V> first = entryAt(tab, index);

        // 下面这串 for 循环虽然很长，不过也很好理解，想想该位置没有任何元素和已经存在一个链表这两种情况
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        // 覆盖旧值
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                // 继续顺着链表走
                e = e.next;
            }
            else {
                // node 到底是不是 null，这个要看获取锁的过程，不过和这里都没有关系。
                // 如果不为 null，那就直接将它设置为链表表头；如果是null，初始化并设置为链表表头。
                if (node != null)
                    node.setNext(first);
                else
                    // 头插法
                    node = new HashEntry<K,V>(hash, key, value, first);
                // count 为segment当前的node数量
                int c = count + 1;
                // 如果超过了该 segment 的阈值，这个 segment 需要扩容
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node); // 先扩容，在扩容方法里面插入节点
                else
                    // 没有达到阈值，将 node 放到数组 tab 的 index 位置，
                    // 其实就是将新的节点设置成原链表的表头
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        // 解锁
        unlock();
    }
    return oldValue;
}

```

- 先扩容，再插入。
  先找是不是已经存在这个元素，如果存在，那么就可以直接替换，如果找不到就需要数量+1，然后判定阈值，超过阈值那么就 rehash 然后插入新的数组中。

- 并行度
  以 Segment 为单位

## 1.8

### 红黑树链表转换

当 tab 的长度大于等于 64，同时槽中的数量大于 8 就会触发树化

而移除的时候，如果树的长度太小，就会触发树转链表。当扩容的时候，一个树拆成两个的时候，会根据 size 是否大于 6 选择是否链表化。

```
else if (t.removeTreeNode(p))
    setTabAt(tab, i, untreeify(t.first));
```

```
// removeTreeNode
if ((r = root) == null || r.right == null || // too small
        (rl = r.left) == null || rl.left == null)
        return true;
```

### 扩容

- 扩容时机

1. 当链表转红黑树的时候，如果容量小于 64，那么就会触发扩容。
2.

ForwardingNode 表示已经移动完了，里面的 nextTable 是完整的 nextTable,但是不知道为什么要放进每个 ForwardingNode 中一份
#### 扩容占据
通过一个int记录剩余位置数量，占据数量通过cas扣减，相当于redis库存扣减。扣减完成后，那么扣减的这个段就是当前线程负责的段，进行迁移即可。

#### 如果帮助搬迁的线程已满，其他线程会怎么办？
似乎会陷入无限循环。
# HashMap 头插法和尾插法对比

由于 HashMap 的插入必须要遍历 hash 槽内的链表或者二叉树，因此头插和尾插的性能是一样的，只是看具体实现使用的是哪个。

当然，即使像 jdk1.8 的 ConcurrentHashMap 这种，头结点作为锁的，虽然实现用的是头插法，但是使用尾插法也是一样的。因为对于节点的操作，只有获取锁的节点才能操作，当节点获取锁后，只需要判定当前节点是否是头结点即可，这种情况还能够减少头结点的变化，多次加锁。因为头插法，其他节点也在等待加锁，然后新的节点插入 hash 槽作为头结点，那么头结点就变化了，就要重新加锁。而尾插法的情况，头结点是基本不变化的，只有移除节点时可能导致头节点的变化。

# 同步工具类
## AtomicStampedReference
解决ABA问题，通过Stamp作为数据的版本，cas的方式替换对象引用，这样能够保证原子性
```
 AtomicStampedReference<Integer> reference = new AtomicStampedReference<Integer>(1);
        int stamp = reference.getStamp();
        reference.attemptStamp(2, stamp + 1);

public boolean attemptStamp(V expectedReference, int newStamp) {
        Pair<V> current = pair;
        return
            expectedReference == current.reference &&
            (newStamp == current.stamp ||
             casPair(current, Pair.of(expectedReference, newStamp)));
    }

private boolean casPair(Pair<V> cmp, Pair<V> val) {
    return UNSAFE.compareAndSwapObject(this, pairOffset, cmp, val);
}
```
## 为什么cas能够提高性能
对于计算密集型的操作，临界区的占据时间是非常短的，只需要一个简单的赋值即可，但如果加锁的话，加锁的成本可能都高出操作本身很多，因此通过cas能够非常低成本的进行数据的变更，唯一的缺点就是大量的竞争的情况下可能有大量的重试，但是因为本身cas的性能好过加锁，因此操作的吞吐量远远大于加锁。

# 锁总结
## synchronized
- 非公平
- 1.4引入自旋锁
- 1.6引入适应性自旋锁，并默认开启
- 1.6引入偏向锁，用于应对总是一个线程反复获取锁的场景，但是当hashcode被调用，偏向锁会失效(没有位置存偏向锁的数据)
- 锁消除和锁粗化除了可以对于我们编写的不好的同步代码，也能**优化同步容器的操作**或者其他带了同步的操作。
-当轻量级锁已经被获取，其他线程cas获取锁失败(自旋获取)会升级为重量级锁，而原轻量级锁解锁的时候，会解锁失败，其会解锁重量级锁并更新锁的状态唤醒其他竞争线程。

## 为什么要使用volatile
- 禁止重排序
对于对象，禁止构造方法初始化与引用赋值重排序
- 可见性
将数据写入主存，并从主存

## final禁止重排序
对于final字段的写，禁止重排到构造方法外。
对于final字段的读，禁止与读引用重排序。

效果就是对象初始化后，读去final字段一定能够拿到值。而如果是常规字段，那么就可能拿不到