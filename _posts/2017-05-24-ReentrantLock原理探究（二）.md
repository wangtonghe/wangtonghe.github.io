---
layout: post
title:  "ReentrantLock原理探究（二）"
date:   2017-05-24 12:52:00 +0800
categories: java AQS 并发与多线程
header-img: img/posts/java/reentrantLock-cover.jpg
tags:
 - java
 - 并发与多线程
 - AQS
---


## 前言

上篇[ReentrantLock原理探究(一)](http://blog.csdn.net/wthfeng/article/details/72510804)介绍了ReentrantLock类的使用说明，详细解析了关于非公平锁的`lock()`过程。这篇我们继续分析。


## 三、源码解析

### 2.unlock()方法

公平锁与非公平锁的`unlock()`方法相同，就不用区别了。`unlock()`方法调用了`release()`方法。

```java
  public final boolean release(int arg) {
        //尝试释放锁
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
看看 `tryRelease()`,这个方法实现在`ReentrantLock`的内部类`Sync`而不是分别留在`FairSync`或`NonfairSync`中，这也说明了释放锁的过程与锁的公平性无关。

```java
 protected final boolean tryRelease(int releases) {
            //每次释放锁状态减1，若为0说明锁已释放完毕
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            //若锁已释放，将表示占有锁的线程变量设为null
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
  }

```
这个方法比较简单，仅仅是检查并设置状态。不过仍需注意的是，释放锁的方法必须由占有锁的线程调用，否则，则会抛出`IllegalMonitorStateException`异常。

假设线程1已执行完，调用`unlock()`执行到此，会将表示阻塞线程个数状态的`state`设置0。返回`true`,进if语句。

```java
 Node h = head;
 if (h != null && h.waitStatus != 0)
     unparkSuccessor(h);
 return true;

```
此时`head`节点是没有线程的空节点，其后跟着表示线程2的线程节点。而`head`节点的`waitStatus`为`SIGNAL `,值为-1。

进入`unparkSuccessor(h)`方法,`h`是队列的头节点。

```java
    private void unparkSuccessor(Node node) {
        /*
          若节点`waitStatus`为负值，设置0
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
          这里有JDK注释，大意是，解锁阻塞队列中的线程。一般情况下是下一个节点（head后面的节点），但若这个节点表示的线程被取消了，则往后遍历直到找到需释放的（waitStatus为负值的）。
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        //解锁head节点后的第一个等待节点
        if (s != null)
            LockSupport.unpark(s.thread);
    }

```
`LockSupport.unpark(s.thread)`方法应该不陌生，上篇中就是用`LockSupport.park(this)`方法将当前线程锁住的，底层调用的是`Unsafe`类方法。这里暂不研究。

这里和`lock()`的过程对接一下，当线程2完成`unlock()`过程。唤起线程1。

此时线程1在`acquireQueued()`的循环中,

```java
          for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }


```
走循环的第一个if,执行完`tryAcquire()`后线程2已获取锁。这里进if,设置头节点

```java
  private void setHead(Node node) {
        head = node;
        node.thread = null;
        node.prev = null;
    }
```
经过设置，阻塞队列重新恢复成只有一个空的头节点的状态。这样获取、释放锁的流程就走完了。其他类似。


### 3. 公平锁的获取方式

公平锁与非公平锁的解锁流程是一致的，区别只在锁的过程，即`FairSync`的`tryAcquire`。

```java
     protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

```
公平锁比非公平的只在`hasQueuedPredecessors()`方法。此方法会判阻塞队列前面是否有其他线程在等待。

```java
  public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }

```
其他步骤与非公平锁一致。

### 4. 线程取消

前面一直提到线程取消，这里解析一下

在`acquireQueued()`方法中`finally`代码段是处理异常发送后取消线程的，主要调用了`cancelAcquire(node)`。node为要取消的线程表示的节点。

```java

    private void cancelAcquire(Node node) {
        //节点为空直接返回
        if (node == null)
            return;

        node.thread = null;

        // 跳过被取消的（即waitStatus大于0）节点
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

        //node前置节点的后置节点，不一定是node
        Node predNext = pred.next;

        // 将节点状态改为取消状态
        node.waitStatus = Node.CANCELLED;

        //若节点是尾节点，直接设为null
        if (node == tail && compareAndSetTail(node, pred)) {
            compareAndSetNext(pred, predNext, null);
        } else {
            //前置节点不是head,表明该节点前有阻塞的线程
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    //连接该节点的前置节点和后续节点，即去掉该节点
                    compareAndSetNext(pred, predNext, next);
            } else { //前置节点是head
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }

```
注释都说差不多了，最后调用的`unparkSuccessor`前面解锁过程已经提过。可以发现，如果外界有异常导致当前线程被取消时，会设置状态，并根据该节点所在位置恰当退出队列。


## 四、Condition研究

上篇示例中演示了`Condition()`的用法，这里简要分析一下它的底层。

### 1. 类结构
AQS内部类`ConditionObject`实现了`Condition`接口，主要有下列两个字段.

```java
  //条件队列第一个节点
  private transient Node firstWaiter;
  
  //条件队列最后一个节点     
  private transient Node lastWaiter;
  

```

### 2.await()方法

`await()`方法需结合Lock类使用，上篇示例中已经提过，此方法与`Object.wait()`功能相近，阻塞当前线程，直到调用相同锁的`signal()`方法。

`await()`方法有众多版本，如`awaitNanos(long nanosTimeout)、awaitUntil(Date date)、await(long time, TimeUnit unit)`。这里以`await()`为例。

> 为表述方便，我们以[Condition测试用例](https://github.com/wangtonghe/learn-sample/blob/master/learn/src/java/com/wthfeng/learn/thread/ConditionTest.java) 为实例，有线程A、B两个线程。线程A遍历到20调用`await()`阻塞，线程B休眠2秒唤醒线程A。

```java
   public final void await() throws InterruptedException {
            //显式检查中断
            if (Thread.interrupted())
                throw new InterruptedException();
            //用当前线程构造等待条件节点
            Node node = addConditionWaiter();
            //当前线程释放锁
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            //如果不在阻塞队列中，将自己锁住
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            //以下是被唤醒后重新竞争锁，与lock过程类似
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
    }

```
> `await()`方法为暂停该线程，等待唤醒。线程A调用后会阻塞在上面的`while`循环中。等待唤醒。


首先会检查中断标志，这也是`await()`方法会响应中断的实现点。`addConditionWaiter()`以当前线程为节点，`fullyRelease()`释放当前线程持有的锁，内部调用的是`release()`。再判断节点是否在队列中，若不在，在循环中阻塞该线程。循环后面的语句是经`signal()`唤醒后竞争锁的过程。与`lock`就关联起来了。


### 3.signal()方法
`signal()`主要作用就是唤醒被`await`的线程。与`await()`一样，该方法必须在获取锁的情况下使用。

```
  public final void signal() {
       //当前线程是否是锁的持有者
       if (!isHeldExclusively())
           throw new IllegalMonitorStateException();
       Node first = firstWaiter;
       if (first != null)
           doSignal(first);
  }
```
主要处理逻辑在` doSignal(first)`，`first`是条件队列的首节点。

```java
   private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
             //遍历找到第一个可以释放的节点（不能是被取消的）
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
   }

```
看看`transferForSignal(first)`

```java
    final boolean transferForSignal(Node node) {
        //
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        // 将要唤醒的线程节点加入阻塞队列等待唤醒
        Node p = enq(node);
        int ws = p.waitStatus;
        //该节点被取消，直接释放解锁
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }

```

总结一下`signal()` 的流程。

线程A已被阻塞（处于`while`循环），此时在等待队列中（由Condition维护，不同于AQS的同步阻塞队列）。线程B调用`signal()`后线程A从等待队列取出，放到同步阻塞队列。此时正好破解了线程A的while循环。

```java
// await()方法

 while (!isOnSyncQueue(node))  //是否处于阻塞队列

```
这样一来流程就清晰了。等线程B执行完后（执行完`unlock()`释放锁），由于同步阻塞队列没有其他阻塞线程，线程A就可获取锁继续执行了。

相反，假设阻塞队列有排队的线程，即不止一个线程（这里只有线程A）等待释放，走`await()`后面的竞争锁过程，其实是`lock`阻塞的过程，这里就不提了。



## 参考文章

1. [怎么理解Condition](http://ifeve.com/understand-condition/)
2. [ReentrantLock实现原理深入探究
](http://www.cnblogs.com/xrq730/p/4979021.html)
3. [AbstractQueuedSynchronizer的介绍和原理分析](http://ifeve.com/introduce-abstractqueuedsynchronizer/)
4. [ Java线程(九)：Condition-线程通信更高效的方式](http://blog.csdn.net/ghsau/article/details/7481142)




















































