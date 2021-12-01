---
title: "Java高并发笔记" # 标题
date: 2021-12-02T01:31:06+08:00    # 创建时间
lastmod: 2021-12-02T01:31:06+08:00 # 最后修改时间
tags: ['并发']
categories: ['并发']  # 分类
---

# 并发

## 概念

死锁：堵死了，无法结束

活锁：动死了，互相续命，无法结束

无障碍：都进入临界区，可不导致一方堵塞。如：fail safe情况规范运行

无锁：100个保证有一个可以成功，每次临界区内有一个成功，是属于一种无障碍锁。如：cas

无等待：临界区内都可以成功。如RUC（readcopywrite），一种读无等待锁

有序性：因为程序只保证串行语义一致，而不保证多线程语义

### **同步与异步**

- **同步：** 同步就是发起一个调用后，被调用者未处理完请求之前，调用者**等待被调用者返回**。
- **异步：** 异步就是发起一个调用后，**立刻得到被调用者的回应**（如表示已接收到请求），但是被调用者并没有返回结果，此时我们可以处理其他的请求，被调用者通常依靠事件，**回调等机制来通知调用者其返回结果**。

### **堵塞和非堵塞**

形容多线程的相互影响，**如堵塞等待的时候线程挂起**

- **阻塞：** 阻塞就是发起一个请求，**调用者一直等待请求结果返回**，也就是当前线程会被挂起，无法从事其他任务，只有当条件就绪才能继续。
- **非阻塞：** 非阻塞就是发起一个请求，调用者不用一直等着结果返回，**可以先去干其他事情**，需要的时候再去问。

### 堵塞和同/异步

**关注点不同**

同步是主动调用，异步是被动调用（调用可换成使用/获取，返回方式），如程序的系统调用（中断/协程是异步，运行是同步）

- 同步和异步，指的都是**应用程序级的**。

堵塞和非堵塞，描述的是程序是否不动了，针对的是**是否立即响应，能否做其他**

- 阻塞和非阻塞，在进程状态的概念，如在内核调度进程是否需要等待
  - 堵塞等待cpu返回：**一些功能调用**（一般等待成本高于切换成本）或者NIO（堵塞等几秒/等全部结果）或者传统IO
    - 所以相对的来说非堵塞一般是乐观的，堵塞是悲观的
  - 不堵塞cpu立即返回：NIO（不堵塞立即返回结果情况），非阻塞I/O 系统调用
    - **异步**I/O系统调用**都是非阻塞式**的行为
- 堵塞和非堵塞一般是针对IO进行描述的，因为多线程的优势之一就是解决IO问题

> NIO为什么可以堵塞可以非堵塞？

- 对于NIO是同步非堵塞IO 拿到的可以是不完整的，而异步非堵塞I/O系统调用 read()结果必须是完整的
  - 对IO来说,简单说同异步是**返回内容要求的区别**

**概念上交叉**

在进程通信层面，**阻塞/非阻塞，同步/异步基本是同义词**

同步堵塞：NIO（IO层面，可选择堵塞，内容完整），普通IO

同步非堵塞：NIO（IO层面）

异步非堵塞：协程（进程层面）

[堵塞和非堵塞]: https://leetcode-cn.com/circle/discuss/uHGOZo/
[NIO是否堵塞]: https://github.com/CyC2018/CS-Notes/issues/194
## 并发级别

并发级别

1. 堵塞，挂起等IO
2. 无饥饿：FIO，公平
3. 无障碍：运行并发，如果冲突就回滚，Fail-safe原则
4. 无锁：CAS
5. 无等待：读写分离，ReadCopyUpdate

并发概念

1. 原子性：改变是否是数据一致安全的
2. 可见性：缓存是否同步的
3. 有序性：缓存的改变顺序，JVM的指令流水线优化
   1. HappendBefore原则可以保证

**线程状态**

协助线程的方法

wait和notify和sleep和yield和join（t1.join 原理是while(isAlive())wait()）和守护线程（守护线程随主线程结束）

> 使用wait的对象必须是syn(obj)的对象 // 不要求在syn内，可以在syn内的调用方法内
>
> syn(类锁)
>
> wait会导致重量级锁

```JAVA
// 线程的几个状态 由线程额
// join原理是wait  wait原理是park
public static State toThreadState(int var0) {
        if ((var0 & 4) != 0) {
            return State.RUNNABLE; // 正常的运行态。挂起的时候(resume)可是已经让出CP，已经废弃
        } else if ((var0 & 1024) != 0) {
            return State.BLOCKED; //  synchronize  //Monitor也  
        } else if ((var0 & 16) != 0) {
            return State.WAITING; // park堵塞  注意重入锁的lock原理是park所以也是wait
        } else if ((var0 & 32) != 0) {
            return State.TIMED_WAITING;// park(100)  sleep(1000)
        } else if ((var0 & 2) != 0) {
            return State.TERMINATED; // over ,start() 后 isAlive()== false
        } else {
            return (var0 & 1) == 0 ? State.NEW : State.RUNNABLE; //NEW：无start状态
        }
    }
```

![preview](https://pic3.zhimg.com/v2-287f87ad5328f2aa5cd7fbd48dadcd8f_r.jpg?source=1940ef5c)

**线程组和线程优先级**

```java
ThreadGroup tg = new ThreadGroup("Group"); //创建线程组
Runnable r = new Runnable();
Thread t = new Thread(tg,r, "t1")
	t.setDaemon(true);// 守护线程
	t.setPriority() //优先级 越大越高，1-10， 
	t.getThreadGroup()
    tg.list();
    tg.activeCount();
```

**并行要注意的问题**

1. 线程不报错，无异常日志，难以查找
2. 使用并发不安全集合ArrayList数据丢失
3. jdk8之前并发hashmap会有死锁问题
4. 错误的加锁方式：对非对象/类进行加锁，对装箱对象加锁（无意识的丢失原对象）
5.

## JUC工具

jdk的并发包，增加了同步控制工具

**重入锁**

ReentrantLock在jdk5之前优于synchronize，可重入锁必须在lock后unlock

```java
Main.class.wait(); // 会导致lock的编译unlock失效？
ReentrantLock lock = new ReentrantLock();
lock.lock();
newCondition可以实现多等待队列，通过awiat和signal
lockInterruptibly() 可中断的
trylock()  可尝试的
trylocak(100) 可时间循环获取的 

```

Condition

```java
public final void signal() {
    if (!isHeldExclusively())  // Condition和锁AQS是绑定的，如isHeldExclusively方法就属于AQS
}
```

**Semaphore**

简化锁，进行信号量控制

```java
Semaphore semp  = new Semaphore(6);
semp.acquire()//抢1
semp.release()//释放1
```

**ReadWriteLock 读写锁**

采用同一个AQS，双AQS子类构成独写.

```java
ReentrantReadWriteLock readWriteLock = new ReetrantReadWriteLock()
Lock readLock = readWriteLock.readLock() //重叠读，不可跌写
Lock writeLock = readWriteLock.writeLock()//互斥
    // 用法和ReentrantLock基本一样， 同个AQS子类所以Condition队列是同样合适的
    readLock.lock()()
```

注意普通独写锁可能导致饥饿，StampedLock可以防饥饿（采用乐观锁，出现写锁而不强占）

**CountDownLatch**

通常可以充当检查任务的栅栏，基于AQS的共享锁机制

```java
CountDownLatch end = new CountDownLatch(10); // 通过初始化10次的共享锁 //类似读
end.countDown(); //每次countDownt进行解除一个共享
//----临界线，调用await等待完成
// int tryAcquireShared(int acquires) { return (getState() == 0) ? 1 : -1; } 只要不是0就获取共享失败，所以读堵塞
end.await();//获取个共享锁，进行堵塞等待，等待一个sign唤醒。  因为当10个任务（线程）完成的时候，共享解除，释放堵塞的锁进行unpark

```

**CyclicBarrier**

循环栅栏，不是直接基于AQS，而是基于ReentrantLock

相比CountDown多接收了一个BarrierRUn  => Runnable  ，每次检查任务结束就**运行一次BarrierRun**,并且将**await的放行**，可以通过改变Barriuer的状态进行编程(逻辑应该集成在BarrierRun)，如果结合状态模式会有很好的扩展性。

```java
CyclicBarrier cycLic  = new CyclicBarrier(10,new BarrierRun) // 检查个数和BarrierRUn  
cyclic.await() //堵塞
// 每10个await进行调用一次command.run() =》  运行BarrierRun(),
// nextGeneration()/breakBarrier();=》trip.signalAll();  count = parties; 重置count循环计数 
```

**LockSupport**

不会抛出中断异常需要自己进行逻辑处理，一般配合Thread.interrupted进行中断恢复，并且通过返回值判断是正常的unpark还是非正常的中断

> 注意park方法底层判断是中断，是不会进行堵塞的

为了兼容wait，所以**采用park(Object)的方法**进行对锁，保持一致，分析问题更加简单。

还有就是park的线程状态是wait，所以要区别开。

## 线程池

Exector框架帮助开发人员有效进行线程控制，也减少了造轮子。

> 详见Java的线程池演化

```java
Executors.newFixedThreadPool(10) // 固定多少个  n,n,LinkedBlockingQueue()
newSingleThreadExecuor() //单个 1,1,LinkedBlockingQueue()
newCachedThreadPool()//可复用池，高峰池     0,max,60L,SynchronousQueue
newSingleThreadScheduledExecutor()// 单个定时/周期
// 可以指定数量
Executors.newScheduledThreadPool(10)
// Rate是以上个任务开始（容易太多在一起），Delay是以上个任务结束（时间不精确），异常都会中断
ses.scheduleAtFixedRate(new Runnable(){},0,2,TimeUnit.SECOUNDS) //0s开始，每2s执行一次
```

线程池化基本都是由ThreadPoolExecutor进行封装创建的，原理是堵塞队列+数量限制

```java
// newFixedThreadPool
new ThreadPoolExecutor(nThreads,nThreads,0L,TimeUnit.MILLISECOUNDS,new LinkedBlockingQueue<Runnable>));
// newCachedThreadPool
new ThreadPoolExecutor(
    0, // corePoolSize 线程数量复用上限
    IntMAX, //最大的线程数量
    60L, // 数量超corePoolSize，的存活时间
    TimeUnit.MILLISECOUNDS, // 时间单位
    new SynchronousQueue<Runnable>() //任务队列
    //  默认就好，线程工厂，创建线程
    //  拒绝策略
    );
```

ThreadPoolExecutor采用的任务队列

```java
// 直接提交队列
// 非队列保存线程，超PoolSize就拒绝。
//（采用匹配模式，每次put等待一次匹配take，put等take，take等put，使用transfer方法进行处理匹配队列）
SynchronousQueue
//有界队列，有coreSize，等待队列有限，超PoolSize就拒绝
ArrayBlockingQueue
//无界队列，只有coreSize，有等待队列无限，超可以一直堵塞。
//（LinkedBlockingQueue 可以设置大小那就有限制了）
//优先队列，调度优先， 属于无界队列 
 LinkedBlockingQueue 
```

ThreadPoolExecutor和任务队列的**搭配流程**

- 原理是先看线程池有没有空闲的线程，由corePoolSize控制（超过corePoolSize，maxPoolSize和过期时间控制，还没过期的线程）
  - 如果是同步队列，没有corePoolSize() 所以只受maxPoolSize和过期时间控制，所以合适做高峰线程
- 如果没有空闲线程就会加入队列（根据队列的特性变化）
  - 有界队列满了，就选择**控制maxPoolSize以下开启线程**（FIFO），如果**超了就拒绝策略**
  - 无界就一直放在队列不管了，maxPoolSize只要**大于corePoolSize就好无特别大意义**

```java
//简单调用
ExecutorService s = Executors.newSingleThreadExecutor();
s.submit(()->{
    System.out.println("sadadas");
});
```

**原理流程源码**

```java
// ThreadPoolExecutor.excute源码
if (command == null)
    throw new NullPointerException();
// 拿现在线程的编号
int c = ctl.get();
// workerCountOf当前线程数
if (workerCountOf(c) < corePoolSize) {
    // 直接newWorker 创建线程进行工作
    if (addWorker(command, true))
        return;
    c = ctl.get();
}
// isRunning判断运行数在0以下，因为c的值是-1左移，AtomicInteger递增的，防止超最大值
//这个线程没能直接跑，workQueue.offer进入等待队列
if (isRunning(c) && workQueue.offer(command)) {
    int recheck = ctl.get();
    if (! isRunning(recheck) && remove(command))
        reject(command);
    else if (workerCountOf(recheck) == 0)
        addWorker(null, false);
}
//进入队列失败，如有界队列或者同步队列，那就创建限时线程任务
else if (!addWorker(command, false))
    // 到达poolsize那就拒绝策略
    reject(command);
```

![img](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9oRXgwM2NGZ1VzWEFqNk9yVVRVRFJvRzV0Q0JnbTRDSmliRmFVc1c1WWJnT1RyN0dFb1JQZWtxOU5xdm5HWTkyYmlhTUpvZHBaTUZtQTFtWnRnQUticE1BLzY0MA)

### **自定义线程池**

#### **可选操作**

```java
new ThreadPoolExecutor(5,5,0,TimeUnit.MiILLISECONDS),
// 可以操作日志
new LinkedBlockingQueue<Runnable>(10){
     @Override
    beforExecutor(Thread t,Runnable r){} //执行
    @Override
    afterExecutor(Thread t,Runnable r){}..... //完成
	 @Override
     terminated().... // 线程池退出
},
//Executors.defaultThreadFactor<>(),
// 初始化工作，如守护线程
new ThreadFactory(){
    @Override
    public Thread newThread<Runnable>(){
        return new Thread();
    }
},
// 可以拒绝日志等记录
// new AbortPolict()
new RejectedExecutionHandler(){
    @Override
    public void rejectExexcutor(Runnable r ThreadPoolExcutor executor){
        System.out.println("OMG");
    }
}
```

#### **线程池的数量**

估值效率的公式：`Nthreads= Ncpu*Ucpu*(1+W/C)`

Java获取, Cpu数量：Runtime.getRuntime().availableProcessors()

Ucpu是cpu利用率，Ncpu是cpu核数，W/C：是等待时间与计算时间的比例

### 线程池错误的处理

当**submit() 任务抛出错误的时候会丢失错误**，可会导致线程池执行中的任务全部结束，从而产生异常堆栈

采用execute() 或者Future re = pools.submit()  re.get() ，可以保留堆栈信息，可是**只能获取线程内的堆栈，而不知道在哪调用的**

为了获取调用堆栈，可以通过**封装Runnable** 对调用的地方再 `new Excetion()`从而达到保存堆栈的目的

## **Fork/Join框架**

```java
ForkJoinPool forkJoinPool = new ForkJoinPool();
//CustTask extends RecursiveTask<T> {  public T compute(){return T};}
    // compute内进行fork分解任务，一个comPute属于大任务，里面的fork都是分解任务，以此递归
    // CustTask.fork() =>提交任务（fork当前线程，也就由当前线程所属线程池管理） 
    // CustTask.join()=>  获取调用compute的返回值
// forkJoinPool.submit(CustTask) =>提交任务到池为核心然后
// forkJoinPool.get(=> 获取调用compute的返回值  ForkJoinPool线程池，没有返回会堵塞
ForkJoinTask<T> result = forkJoinPool.submit(task); 

```

## 集合类的并发

Collections.synchronizedMap() 采用Syn(mutex)进行锁

ConcurrentLinkedQueue() 高并发链表，采用懒双链表进行CAS拼接的方式，每次都casNext进行更新，保证拼接的正确性

```java
public boolean offer(E e) {
        checkNotNull(e);
        final Node<E> newNode = new Node<E>(e);

        for (Node<E> t = tail, p = t;;) {
            Node<E> q = p.next;
            if (q == null) {
                // p is last node 
                if (p.casNext(null, newNode)) {
                    // Successful CAS is the linearization point
                    // for e to become an element of this queue,
                    // and for newNode to become "live".
                    // p和t初始化是tail,所以1个时候不更新尾巴，达到2个的时候tail和就和最后一个不一样了再更新
                    if (p != t) // hop two nodes at a time
                        // 
                        casTail(t, newNode);  // Failure is OK.
                    return true;
                }
                // Lost CAS race to another thread; re-read next
            }
            else if (p == q)
                // We have fallen off list.  If tail is unchanged, it
                // will also be off-list, in which case we need to
                // jump to head, from which all live nodes are always
                // reachable.  Else the new tail is a better bet.
                p = (t != (t = tail)) ? t : head;
            else
                // Check for tail updates after two hops.
                p = (p != t && t != (t = tail)) ? t : q;
        }
    }
```


[书籍]: 《Java高并发程序设计》
