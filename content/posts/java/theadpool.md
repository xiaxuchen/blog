---
date: 2024-03-04T23:09:59+08:00
title: "Java线程池详解"
summary: "线程池是如何工作的？线程池的源码解析"
series: ["Java源码"]
weight: 5
tags: ["Java"]
author: ["夏旭晨"]
---

# ThreadPool 执行流程

1. 如果核心线程池没满，那么直接创建核心线程池去执行
2. 如果核心线程池满了，那么尝试将任务丢进队列中
3. 如果丢进队列失败了，则尝试创建 worker 线程直接执行任务
4. 如果 worker 无法创建，那么就会 reject

# 为什么线程池 Worker 不会把核心 Worker 删掉

```
// ThreadPoolExecutor.Worker.getTask
private Runnable getTask() {
    // 如果这个方法返回空，那么Woker的run方法中的runWorker方法的循环就会破掉，然后线程就停止执行了
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        // ... 省略部分代码
        // 第一个是如果当前worker都超过最大线程了，就尝试将当前worker关闭
        // 第二个是定时获取任务，并获取超时了，那么就尝试将worker关闭
        if ((wc > maximumPoolSize || (timed && timedOut))
                    && (wc > 1 || workQueue.isEmpty())) {
            // 先扣减线程数量(cas)，会控制不小于核心线程的数量
            if (compareAndDecrementWorkerCount(c))
                // 停止该worker
                return null;
            continue;
        }
        // 如果核心线程是能超时或者当前worker数量大于核心线程数，那么后续拉取任务就可以定时keepAliveTime,如果没拉倒就尝试销毁
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        try {
            // 这里根据是否当前线程数大于
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```
