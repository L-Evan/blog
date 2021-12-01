---
title: "AQS" # 标题
date: 2021-10-31T15:58:01+08:00    # 创建时间
lastmod: 2021-10-31T15:58:01+08:00 # 最后修改时间
tags: ['并发']
categories: ['并发']  # 分类
---
## AQS

## AQS的概述

让**JUC的锁**和同步工具的一个集成框架，抽象队列（AbstractQueueSynchronizer）。

只要堵塞锁相关的JUC同步工具基本都存在

**基于AQS构建同步器：**

- ReentrantLock（可重入锁）
- Semaphore（）
- CountDownLatch
- ReentrantReadWriteLock
- SynchronusQueue
- FutureTask

**优势**

- AQS 解决了在实现同步器时涉及的**大量细节问**题，例如自定义**标准同步状态、FIFO 同步队列**。
- 基于 AQS 来构建同步器可以带来很多好处。它不仅能够极大地**减少实现工作**，而且也**不必处理在多个位置上发生的竞争问题**。

## 核心概念

### State

一个Aqs实现只能有一种模式，state记录**当前aqs模式**的情况（所以读写锁也需要2个AQS实现）

![image-20211014141616521](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211014141616521.png)

### 队列

队列都是一种FIFO的队列

**Sync queue**

属于同步队列，是一个双向列表。包括head节点和tail节点。head节点主要用作后续的调度。

- AQS的核心，head和tail维护的waitStatus的双向链表结构，如ReentrantLock的抢占队列

![image-20211014142124015](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211014142124015.png)

**Condition queue**

非必须，单向列表。当程序中存在condition的时候才会存在此列表。

- ConditionObject作为AQS的内部类，由FirstWaiter存储堵塞的wait队列，如Condition的释放队列

![image-20211014142110112](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211014142110112.png)

**源码位置**

```java
AQS{  
    // 作为节点可以在condition队列，也可在syn队列
    static final class Node {
        	/**
             * Head of the wait queue, lazily initialized.  Except for
             * initialization, it is modified only via method setHead.  Note:
             * If head exists, its waitStatus is guaranteed not to be
             * CANCELLED.
             */
            private transient volatile Node head;
            /**
             * Tail of the wait queue, lazily initialized.  Modified only via
             * method enq to add new wait node.
             */
            private transient volatile Node tail;
        	// 代表节点类型
        	volatile int waitStatus;
      
    }
   	// ------------------------------Syn队列---------------------------------------
    /**
     * Head of the wait queue, lazily initialized.  Except for
     * initialization, it is modified only via method setHead.  Note:
     * If head exists, its waitStatus is guaranteed not to be
     * CANCELLED.
     */
    private transient volatile Node head;

    /**
     * Tail of the wait queue, lazily initialized.  Modified only via
     * method enq to add new wait node.
     */
    private transient volatile Node tail;
    /**
     * The synchronization state.
     */
    private volatile int state;
    // Condition队列
    public class ConditionObject implements Condition{
        /** First node of condition queue. */
        private transient Node firstWaiter;
        /** Last node of condition queue. */
        private transient Node lastWaiter;
    }
}

```

## Aqs方法简介

### 方法

- **setState**(int newState)：设置当前**同步状态**；
  - int 是当前状态值，默认0，**正数一般是重入的次数**
- **compareAndSetState**(int expect, int update)：使用CAS设置当前状态，该方法能够保证状态设置的原子性；
  - Cas的去改变状态值，因为竞争不过会加入队列，实际上竞争不是特别大？
- tryAcquire(int arg)：独占式获取同步状态，获取同步状态成功后，其他线程需要等待该线程释放同步状态才能获取同步状态
  - acquire会调用tryAcquire尝试获取锁，state+1
- tryRelease(int arg)：独占式释放同步状态；
- **tryAcquireShared**(int arg)：共享式获取同步状态，返回值大于等于0则表示获取成功，否则获取失败；
  - 特别是读写锁的**读锁需要共享锁**的概念
- tryReleaseShared(int arg)：共享式释放同步状态；
- **isHeldExclusively**()：当前同步器是否在独占式模式下被线程占用，一般该方法表示是否被当前线程所独占；
  - **condition需要判断是否占用进行释放**
- **acquire**(int arg)：独占式获取同步状态，如果当前线程获取同步状态成功，则由该方法返回，否则，将会进入同步队列等待，该方法将会调用可重写的tryAcquire(int arg)方法；
  - 独占在ReenLock的lock方法有体现
- **acquireInterruptibly**(int arg)：与acquire(int arg)相同，但是**该方法响应中断**，当前线程为获取到同步状态而进入到同步队列中，如果当前线程被中断，则该方法会抛出InterruptedException异常并返回；
- **tryAcquireNanos**(int arg,long nanos)：超时获取同步状态，如果当前线程在**nanos时间内没有获取到同步状态，那么将会返回false**，已经获取则返回true；
- acquireShared(int arg)：共享式获取同步状态，如果当前线程未获取到同步状态，将会进入同步队列等待，与独占式的主要区别是在同一时刻可以有多个线程获取到同步状态；
- acquireSharedInterruptibly(int arg)：共享式获取同步状态，响应中断；
- tryAcquireSharedNanos(int arg, long nanosTimeout)：共享式获取同步状态，增加超时限制；
- release(int arg)：独占式释放同步状态，该方法会在释放同步状态之后，将同步队列中第一个节点包含的线程唤醒；
- releaseShared(int arg)：共享式释放同步状态；

## AQS的抽象方法

作为保证各个锁的特性，**根据自己特性去重写**，也就是尝试加锁和解锁的共享/独占锁，还有就是判断**当前线程是否占有锁**

![image-20211011214720909](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211011214720909.png)

isHeldExclusively

- **condition**才需要去实现它，需要判断占有锁情况，由Condition**内部调用外部实现**

## 详细介绍

### AQS.statte

作为AQS的状态，代表的是当前锁对象的状态

默认**statue 为 0无锁**，锁住为1，重入锁每次statue+1，而对应每次释放锁state-1（wait的时候也-1）

- 重入时，当锁的数量为负数的时候，代表超过重入最大的数量（int溢出）

```java
if (++c < 0) // overflow
    throw new Error("Maximum lock count exceeded");
```

**Node.status**

节点代表着线程的状态，用来区分在condition和在syn队列中的状态

> **CANCELLED**
> waitStatus值为1时表示该线程节点已释放（超时、中断），已取消的节点不会再阻塞。
>
> 这个已经没用了，被中断后已经开始过了

> **SIGNAL**
> waitStatus为-1时表示该线程的后续线程需要阻塞，即只要前置节点释放锁，就会通知标识为 SIGNAL 状态的后续节点的线程
>
> 可以释放了

> **CONDITION**
> waitStatus为-2时，表示该线程在condition队列中阻塞（Condition有使用）

> **PROPAGATE**
> waitStatus为-3时，表示该线程以及后续线程进行无条件传播（CountDownLatch中有使用）共享模式下， PROPAGATE 状态的线程处于可运行状态

一般**负数代表现在为堵塞**，**正数代表已经可以运行**了（或者失效，被中断/计数器到了）

### AQS.acquire

请求锁的方法

```java
// 老版
public final void acquire(int arg) {
    // 尝试获得，没获得加入队列
    if (!tryAcquire(arg))
        // 加入队列
        acquire(null, arg, false, false, false, 0L);
}
// 新版
public final void acquire(int arg) {
    	//先尝试锁
        if (!tryAcquire(arg) &&
            // acquireQueued处理队列元素进行堵塞，park自己  
            // addWaiter 添加到队列尾 并返回node
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            // 自我中断，获取锁失败，堵塞
            selfInterrupt();
 }
// 可中断
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    // 先重置了中断，再尝试直接获取锁，失败了就堵塞
        if (Thread.interrupted() ||
            (!tryAcquire(arg) && acquire(null, arg, false, true, false, 0L) < 0))
            throw new InterruptedException();
    }
---------------------------------------拓展-----------------------------------------------------------------
 /**
     * Main acquire method, invoked by all exported acquire methods.// 主要的方法，独占和共享都由这个处理
     * @param node null unless a reacquiring Condition  //
     * @param arg the acquire argument
     * @param shared true if shared mode else exclusive //是否共享
     * @param interruptible if abort and return negative on interrupt //
     * @param timed if true use timed waits // true 就定时
     * @param time if timed, the System.nanoTime value to timeout
     * @return positive if acquired, 0 if timed out, negative if interrupted  // 定时开，被中断返回负数
     */
    final int acquire(Node node, int arg, boolean shared,
                      boolean interruptible, boolean timed, long time) {
   
    // 添加队列尾部
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
     // 中断自己
	final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            // 注意，在ParkAndCheckInterrupt 被中断唤醒，会继续往下运行，循环进行尝试获取锁
            for (;;) {
                //当前的头节点
                final Node p = node.predecessor();
                // 只有当前头部和node父相同(当前是下个唤醒)尝试抢一次，抢成功证明上一个结束了 =》  下面park堵塞结束后会因为循环再来这
                // tryAcquire子类实现会抛出异常.表示AQS的status值溢出了,或者超出最大值了.也就是failed为true
                if (p == head && tryAcquire(arg)) {
                    // 把当前节点设置为头，因为他抢到了
                    setHead(node);
                    p.next = null; // help GC 链接当前节点的头结点（没用的），直接引用去掉
                    failed = false; // 正常结束，无需concelAcquire
                    // 出口，看是因为什么原因暂停堵塞,对acquire来说，如果是中断那需要重置下中断
                    return interrupted;
                }
                // 抢失败了，该节点的前个节点ws在等（前置node的ws指的是下一个节点的状态）
                //  那这个节点100%需要等。其他情况试试看设置前一个为等待，然后再抢一次
                if (shouldParkAfterFailedAcquire(p, node) &&
                    // 堵塞住自己，且堵塞结束后  锁释放后会重置中断， return 中断状态
                    //（如果是被中断的，interrupted = true，循环重新请求锁）
                    parkAndCheckInterrupt())
                    // 中断停止堵塞的情况
                    interrupted = true;
            }
        } finally {
            // 如果是因为中断failed 为true
            if (failed)
                // 取消节点在队列中的申请状态
                cancelAcquire(node);
        }
    }   
```

## 非AQS抽象方法

initialTryLock： 判断重入锁，每次锁之前测一下

## 尝试实现AQS

**AQS实现注意事项**

- 释放锁时，**setExclusiveOwnerThread应该在state之前**，从而保证ExclusiveOwnerThread可见性

  - exclusiveOwnerThread属于AOS（AQS父类），是不可见的
  - 因为state和exclusive同时保证aqs锁归属，因为state属于aqs，其值只能代表当前锁状态，**不能代表锁的改变**
- state保证可见性，所以需将
- 采用CAS进行锁的加锁解锁，如没固定顺序无需关心ABA问题

  - state是0或者1 只是改变状态

![image-20210812202334744](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210812202334744.png)

![image-20210812203259719](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210812203259719.png)

### 同步队列过程

<img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210813220533026.png" alt="image-20210813220533026" style="zoom:50%;" />

<img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210813222502498.png" alt="image-20210813222502498" style="zoom:50%;" />

如果获得成功，会断开肖兵前线（**setHead**(node) ）同时也是吧**当前节点指向的线程脱离去own**，**node变成新肖兵节点**，值复位为0可以唤醒其他线程。断下条线（**老肖兵的下个节点设为null**）

![image-20210813222451567](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210813222451567.png)

- 释放锁，unlock调用release进行释放是直接唤醒头部（FIFO）

![image-20210813224306517](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210813224306517.png)

### 是否属于公平锁

唤醒都是从等待队列头部唤醒的，一样是顺序的

- 确认**公平与否，并不是队列唤醒是否随机**，而是唤醒时线程**不直接进入队列尾部**，**重新进行tryLock**
  - 意味之如果在这个**间隙，有新的线程尝试加锁，会导致进行竞争**（不公平的）

![image-20210814002136709](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210814002136709.png)

![image-20210814002143650](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210814002143650.png)

对state+1 实现重入

---

关于打断的设计

![image-20210814010412491](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210814010412491.png)

注意，**当锁采用可中断的时候**，在partAndCheckInterrupt 的时候，返回true，也就是**非正常唤醒的时候，进行throw 异常**，进行继续运行，也就是自己激活

非公平锁

![image-20210814011122593](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210814011122593.png)

判断是否有等待的或者，是重入都无需try， **防止竞争**不公平

![image-20210814013908634](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210814013908634.png)


[aqs介绍]: https://juejin.cn/post/6844903997438951437?share_token=cfdd7c79-e6b8-4ded-946f-7d3672ed97f7
[美团技术-从ReentranLock看AQS]: https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html
