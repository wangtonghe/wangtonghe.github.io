---
layout: post
title:  "ReentrantLock原理探究（一）"
date:   2017-05-21 10:21:00 +0800
categories: java AQS 并发与多线程
header-img: img/posts/java/reentrantLock-cover.jpg
tags:
 - java
 - 并发与多线程
 - AQS
---


## 前言

ReentrantLock类是synchronized语义的替代品，可以实现与其相同的功能，了解其实现原理对并发编程无疑是很有帮助的。其次，ReentrantLock 的实现基础AQS(AbstractQueuedSynchronizer)也是Java并发编程中相当重要的一个类，所以无论如何，我们都要了解一番。

## 一. 用法及概念

### 1. 用法

`ReentrantLock`（可重入锁）由java1.5引入，被用来实现`synchronized`关键字语义。这样就可以从代码级别而不是语言层面实现锁的定义。`Lock` 是其接口，规范了若干作为锁必须实现的方法。

下面使用`lock` 来保证`i++` 操作的线程安全。

```java
  private void addUseLock() {
        Lock lock = new ReentrantLock();
        //在进行操作前先锁定
        lock.lock();
        try {
            flag++;
        } finally {
            //操作结束后一定要在finally中释放锁，否则可能导致死锁
            lock.unlock();
        }
  }
```

当然，我们也可以用`synchronized`语义实现

```java
  private synchronized void addUseSync() {
        flag++;
  }
```

`Lock` 可以创建`Condition`(条件变量)，用来实现` wait() 、 notify()`语义。从而控制线程间的通信。`Condition` 中的`await`与`wait`类似，`signal`则相当于`notify`。两种机制使用也很相似，都**必须在获取锁的情况下操作**。

下面模拟了`Condition` 的使用，`CountDownLatch` 用来使主线程等待两个工作线程结束，可以暂不研究。

```java
 @Test
    public void test() {

        Lock lock = new ReentrantLock();
        
        //创建一个条件
        Condition condition = lock.newCondition();

        CountDownLatch countDownLatch = new CountDownLatch(2);

        /**
         * 线程A遍历到20后自我阻塞，等待线程B唤醒
         */
        new Thread(() -> {
            try {
                lock.lock();
                for (int i = 0; i < 100; i++) {
                    System.out.println(Thread.currentThread().getName() + ":" + i);
                    if (i == 20) {
                        condition.await();
                    }
                }
            } catch (InterruptedException e) {
                System.out.println(e.getMessage());
                e.printStackTrace();
            } finally {
                lock.unlock();
            }

            countDownLatch.countDown();

        }, "ThreadA").start();

        /**
         * 线程B休眠2秒后唤醒线程A
         */
        new Thread(() -> {
            try {
                lock.lock();
                TimeUnit.SECONDS.sleep(2);
                System.out.println("唤醒线程");
                condition.signal();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            finally {
                lock.unlock();
            }
            countDownLatch.countDown();

        },"ThreadB").start();

        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

### 2. 公平锁与非公平锁

`ReentrantLock` 有一个很重要的概念就是锁的公平性。**所谓公平锁，就是使线程按照请求锁的顺序依次获得锁**，反之，就是非公平的。也就是说，如果锁是非公平的，有的线程可能一直不断获取锁，而有的线程可能一直获取不到。而公平锁则不会，它会按请求顺序依次分配。

那为什么要设计非公平锁呢？原因是效率问题。处于CPU调度考虑，采用公平锁会消耗一些性能保证线程调度公平和同步，而且对于重复获取锁的操作中，某个线程连续获取锁的概率是很高的，而公平锁则遏制了这一点。所以，如果线程的执行顺序对你的程序不重要的话，最好使用非公平锁。另外，`synchronized`内置锁也是非公平锁。

```java
//默认为非公平锁
Lock lock = new ReentrantLock();

//通过构造函数创建公平锁      
Lock lock2 = new ReentrantLock(true);
    
```

### 3. 可重入性

从名字就可以看出，`ReentrantLock` 是可重入锁。即一个线程获取该锁后可以在后续过程中多次获取该锁，以避免被自己锁死的情况。这个语义`synchronized`也支持。


## 二、类结构解析

下面我们深入`ReentrantLock` 类结构分析一下

### 1.  ReentrantLock类结构

我们刚才提到，`ReentrantLock` 实现了`Lock`接口。另外，它还有3个内部类。分别是`Sync`、`NonfairSync`、`FairSync`。`Sync` 是一个抽象的内部类，代表锁的基本底层实现。后两者分别是对非公平锁和公平锁的实现。下面来看该类的类继承关系和主要方法。

![](http://img.blog.csdn.net/20170519115721880?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3RoZmVuZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图中可以看到上面提到的三个内部类。深入`Sync`类结构，看看它的结构图

![](http://img.blog.csdn.net/20170519125457341?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3RoZmVuZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

在`Sync`类结构中，`AbstractQueuedSynchronizer`类(简称AQS)有着相当重要的地位。不仅如此，AQS其实是整个java并发包的基础。各个并发工具类如`Semaphore、CountDownLatch、ReentrantLock、ReentrantReadWriteLock、FutureTask` 都是根据自己本身需要在AQS基础上做的定制实现。

这样一来，我们就了解了AQS与ReentrantLock的相互关系。这种实现模式其实是**模板模式**的典型应用。父类负责制定流程，留下若干接口交给子类具体实现。

### 2. AQS类概览

AbstractQueuedSynchronizer类结构比较复杂，具体源码工作我们在下面章节分析，这里先简单看看该类主要结构和方法。

AQS的内部类有2个，`ConditionObject`和`Node`。`ConditionObject` 实现`Condition`接口，用于实现前面提到的条件变量`await/singal`。

`Node`为等待队列的节点类。AQS实现依赖一个先进先出的队列，而`Node`类即是这个队列的节点。用于保存阻塞的线程引用和线程状态。

Node类主要字段

```java

 static final class Node {
     //节点状态
     volatile int waitStatus;

     //前置节点
     volatile Node prev;
      
     //后继节点
     volatile Node next;
     
     //该节点保存的线程
     volatile Thread thread;

     //该队列下一个等待者
     Node nextWaiter;
}  
```
说完AQS的两个内部类，下面了解下AQS的主要字段

```java
//队列头结点
private transient volatile Node head;

//队列尾结点
private transient volatile Node tail;

//同步状态，0表示未锁，>0表示某线程锁了多少次
private volatile int state;

//用于保存独占模式下的当前线程
private transient Thread exclusiveOwnerThread;

```

没有实例理解起来比较吃力，先不研究这个，下步解析源码时再解析。先看看AQS主要方法。

> 说明：获取、释放锁的若干方法中，带`Shared`的是共享方式，否则是以独占方式。

```java

 //获取锁
 public final void acquire(int arg);
 public final void acquireShared(int arg);
 
 //释放锁
 public final boolean release(int arg);
 public final boolean releaseShared(int arg);
 
 //获取、设置状态
 protected final int getState();
 protected final void setState(int newState);
 
 //以原子方式设置值
 protected final boolean compareAndSetState(int expect, int update);
 
 private static final boolean compareAndSetWaitStatus(Node node,int expect,int update);
 
 private final boolean compareAndSetHead(Node update);
 
 private final boolean compareAndSetTail(Node expect, Node update);
 
 
 //尝试获取、释放锁（这些方法留在子类实现）
 protected boolean tryAcquire(int arg);
 
 protected int tryAcquireShared(int arg);
 
 protected boolean tryReleaseShared(int arg)
 
 protected boolean tryRelease(int arg);
 
                                                        
```

AQS主要方法如上。以获取锁为例，`acquire()`方法定义了获取锁的主要流程，其中`tryAcquire()`由子类根据需要定制。

```java
 public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
 }

```
说明：

> 1. `ReentrantLock`使用了独占的方式获取释放锁，主要用到了`tryAcquire `、`tryRelease`。
> 
> 2. 方法中的compareAndSetXXX(arg0,arg1)调用底层CAS原义。即为若内存中该值为arg0,则将其设为arg1。这个设置是原子的，不会中断。


## 三、源码解析

终于到解析源码阶段了。

### 1.lock()方法

先分析非公平锁的`lock`方法。

```java
final void lock() {
    if (compareAndSetState(0, 1))
         setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
 }

```
流程很简单，若`state`为0（即当前没有线程争用），将其设为1。同时把当前线程写入`exclusiveOwnerThread`字段，表示该线程独占锁。

否则(此时表明**这个锁已经被某个线程占了，我们假设为线程1，而当前线程为线程2**)，调用`acquire()` 。

>再往下分析前先想想，既然是独占锁且锁已经被占了，那处理流程应该怎样？ 大致流程应该是把该线程放在某个阻塞队列中，等待前面线程执行完后（调用`unlock()`）,通知该线程获取锁去执行。下面来验证一下。

```java
public final void acquire(int arg) {
        if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
}
```
1. 先执行`tryAcquire()`,成功则说明获取锁成功了，不再执行。
2. 若失败执行`acquireQueued()`,执行后返回一个值，用于判断是否需要自中断


流程就这样，在非公平锁中，`tryAcquire()`调用`nonfairTryAcquire()`

```java
 final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            //获取状态
            int c = getState();
            //若为0，表明前面线程1已执行完，当前线程（线程2）执行获取锁操作
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //表明获取锁的线程是当前线程，即锁重入，status即锁重入次数
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

```
如果线程1还持有锁，线程2执行`tryAcquire()`会直接返回`false`,执行`acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`判断条件。

> 按猜想，这时应该把线程2（当前线程）加到一个阻塞队列存起来，等待线程1执行完后再尝试获取锁。观察代码，该执行`addWaiter()`方法了。

```java
  private Node addWaiter(Node mode) {
        //构建当前线程节点
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

```
以线程2（当前线程）创建一个节点，由于此时阻塞队列为空，直接走`enq(node)`入队操作。


> 说明：参数mode表示队列的模式，这里传的值是`Node.EXCLUSIVE`,表示互斥锁，还有一个变量是`SHARED`,表示共享锁。


看看`enq()`方法

```java

    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            // 判断尾节点是否为空，为空表示队列无值
            if (t == null) { // Must initialize
                 //设置头节点
                if (compareAndSetHead(new Node()))
                     //设置尾节点
                    tail = head;
            } else {
                //尾节点不为空，将当前线程节点入队
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }

```
方法里面是个for()循环。此时队列为空，走`if`逻辑,设置队列头、尾节点。然后再循环一次，此时队列不为空，走`else`,将当前线程节点加入队列，注意形成的是双向队列，而且是头节点为空节点的双向队列。

下面该是`acquireQueued()`方法了

```
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //取出当前节点的前一节点，记为p
                final Node p = node.predecessor();
                //若p是头节点，再次尝试获取锁
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //获取失败，执行`shouldParkAfterFailedAcquire()`方法
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
主要是一个循环两个`if`。第一个`if`里,判断当前线程节点前一个节点是否是`head`（因为head节点总指向空节点），若是表明该节点是阻塞队列第一个节点，这时再次尝试获取锁。成功重设队列`head`节点，返回。失败走第二个if。

> 这里涉及`node.predecessor()、setHead(node)`两个方法。`node.predecessor()`返回的是node节点的`prev`节点。`setHead(node)`将node设为`head`节点，`thread、prev`属性置空。

第二个if,执行`shouldParkAfterFailedAcquire()`。

```java
 private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

```
`waitStatus`表示节点状态，有以下几种状态

|属性|值|说明|
|---|---|---|
| CANCELLED|1|表示当前的线程被取消，处于这种状态的Node会被踢出队列，被GC回收|
|SIGNAL|-1|表示当前节点的后继节点表示的线程需要解除阻塞并执行|
| CONDITION|-2|表示这个节点在条件队列中，因为等待某个条件而被阻塞|
| PROPAGATE|-3|使用在共享模式可能处于此状态，表示后续节点能够得以执行|
|初始状态|0|表示当前节点在sync队列中，等待着获取锁。|

此方法中`prev`参数是`node`参数的前置节点。在我们的例子中，prev是`head`节点。因未赋值，`waitStatus `为0，走`compareAndSetWaitStatus(pred, ws, Node.SIGNAL)`, 将prev的`waitStatus`值设为`Node.SIGNAL`,即-1。返回`flase`。

继续循环，获取不到锁后仍走第二个if,此时`shouldParkAfterFailedAcquire()`返回`true`,进`parkAndCheckInterrupt()`。

```java
  private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
  }
```

调用 `LockSupport.park(this)`将当前线程锁住。最终调用的是`Unsafe`类的`park()`方法。`Unsafe`类提供了一些java底层硬件级别的原子操作。可以提供分配内存、修改对象内存位置、挂起恢复线程等操作。我们暂且不研究这个。



篇幅问题，这篇文章先到这里，等待下篇再来分析公平锁与非公平锁以及`unlock`过程。




------------------未完待续------------------------




## 参考资料

1. [JDK 5.0 中更灵活、更具可伸缩性的锁定机制](https://www.ibm.com/developerworks/cn/java/j-jtp10264/)
2. [ReentrantLock实现原理深入探究
](http://www.cnblogs.com/xrq730/p/4979021.html)
3. [AbstractQueuedSynchronizer的介绍和原理分析](http://ifeve.com/introduce-abstractqueuedsynchronizer/)
4. [深入浅出 Java Concurrency (8): 锁机制 part 3](http://www.blogjava.net/xylz/archive/2010/07/07/325410.html)









