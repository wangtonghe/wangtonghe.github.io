---
layout: post
title:  "Wait/Notify通知机制解析"
date:   2017-12-09 17:15:00 +0800
categories: java thread
header-img: img/posts/java/thread/wait-notify-cover.jpg
tags:
- java
- thread
---

# Wait/Notify通知机制解析

## 前言

我们知道，java的wait/notify的通知机制可以用来实现线程间通信。wait表示线程的等待，调用该方法会导致线程阻塞，直至另一线程调用notify或notifyAll方法才可另其继续执行。经典的生产者、消费者模式即是使用wait/notify机制得以完成。在这篇文章中，我们将深入解析这一机制，了解其背后的原理。

## 线程的状态

在了解wait/notify机制前，先熟悉一下java线程的几个生命周期。分别为初始（NEW）、运行(RUNNABLE)、阻塞(BLOCKED)、等待(WAITING)、超时等待(TIMED_WAITING)、终止(TERMINATED)等状态（位于java.lang.Thread.State枚举类中）。

以下是对这几个状态的简要说明，详细说明见该类注释。


|状态名称|说明|
|-----|------|
|NEW|初始状态，线程被构建，但未调用start()方法|
|RUNNABLE|运行状态，调用start()方法后。在java线程中，将操作系统线程的就绪和运行统称运行状态|
|BLOCKED|阻塞状态，线程等待进入synchronized代码块或方法中，等待获取锁|
|WAITING|等待状态，线程可调用wait、join等操作使自己陷入等待状态，并等待其他线程做出特定操作（如notify或中断）|
|TIMED_WAITING|超时等待，线程调用sleep(timeout)、wait(timeout)等操作进入超时等待状态，超时后自行返回|
|TERMINATED|终止状态，线程运行结束|


![](http://blog.wthfeng.com/img/posts/java/thread/java-thread-status.png)



对于以上线程间的状态及转化关系，我们需要知道

1. WAITING(等待状态)和TIMED_WAITING(超时等待)都会令线程进入等待状态，不同的是TIMED_WAITING会在超时后自行返回，而WAITING则需要等待至条件改变。
2. 进入阻塞状态的唯一前提是在等待获取同步锁。java注释说的很明白，只有两种情况可以使线程进入阻塞状态：一是等待进入synchronized块或方法，另一个是在调用wait()方法后重新进入synchronized块或方法。下文会有详细解释。
3. Lock类对于锁的实现不会令线程进入阻塞状态，Lock底层调用LockSupport.park()方法，使线程进入的是等待状态。


## wait/notify用例


让我们先通过一个示例解析

wait()方法可以使线程进入等待状态，而notify()可以使等待的状态唤醒。这样的同步机制十分适合生产者、消费者模式：消费者消费某个资源，而生产者生产该资源。当该资源缺失时，消费者调用wait()方法进行自我阻塞，等待生产者的生产；生产者生产完毕后调用notify/notifyAll()唤醒消费者进行消费。

以下是代码示例，其中flag标志表示资源的有无。

```java

public class ThreadTest {

    static final Object obj = new Object();  //对象锁

    private static boolean flag = false;

    public static void main(String[] args) throws Exception {

        Thread consume = new Thread(new Consume(), "Consume");
        Thread produce = new Thread(new Produce(), "Produce");
        consume.start();
        Thread.sleep(1000);
        produce.start();

        try {
            produce.join();
            consume.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    // 生产者线程
    static class Produce implements Runnable {

        @Override
        public void run() {

            synchronized (obj) {
                System.out.println("进入生产者线程");
                System.out.println("生产");
                try {
                    TimeUnit.MILLISECONDS.sleep(2000);  //模拟生产过程
                    flag = true;
                    obj.notify();  //通知消费者
                    TimeUnit.MILLISECONDS.sleep(1000);  //模拟其他耗时操作
                    System.out.println("退出生产者线程");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    //消费者线程
    static class Consume implements Runnable {

        @Override
        public void run() {
            synchronized (obj) {
                System.out.println("进入消费者线程");
                System.out.println("wait flag 1:" + flag);
                while (!flag) {  //判断条件是否满足，若不满足则等待
                    try {
                        System.out.println("还没生产，进入等待");
                        obj.wait();
                        System.out.println("结束等待");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.println("wait flag 2:" + flag);
                System.out.println("消费");
                System.out.println("退出消费者线程");
            }

        }
    }
}

```
输出结果为：

>进入消费者线程 <br/>
wait flag 1:false <br/>
还没生产，进入等待 <br/>
进入生产者线程 <br/>
生产  <br/>
退出生产者线程 <br/>
结束等待 <br/>
wait flag 2:true <br/>
消费 <br/>
退出消费者线程 <br/>

理解了输出结果的顺序，也就明白了wait/notify的基本用法。有以下几点需要知道：

1. 在示例中没有体现但很重要的是，**wait/notify方法的调用必须处在该对象的锁（Monitor）中，也即，在调用这些方法时首先需要获得该对象的锁。**否则会抛出IllegalMonitorStateException异常。
2. 从输出结果来看，在生产者调用notify()后，消费者并没有立即被唤醒，而是等到生产者退出同步块后才唤醒执行。（这点其实也好理解，synchronized同步方法（块）同一时刻只允许一个线程在里面，生产者不退出，消费者也进不去）
3. 注意，消费者被唤醒后是从wait()方法（被阻塞的地方）后面执行，而不是重新从同步块开始。


## 深入了解

这一节我们探讨wait/notify与线程状态之间的关系。深入了解线程的生命周期。

由前面线程的状态转化图可知，当调用wait()方法后，线程会进入WAITING(等待状态)，后续被notify()后，并没有立即被执行，而是进入等待获取锁的阻塞队列。

![](http://blog.wthfeng.com/img/posts/java/thread/java-wait-notify.png)


对于每个对象来说，都有自己的等待队列和阻塞队列。以前面的生产者、消费者为例，我们拿obj对象作为对象锁，配合图示。内部流程如下

1. 当线程A（消费者）调用wait()方法后，线程A让出锁，自己进入等待状态，同时加入锁对象的等待队列。
2. 线程B（生产者）获取锁后，调用notify方法通知锁对象的等待队列，使得线程A从等待队列进入阻塞队列。
3. 线程A进入阻塞队列后，直至线程B释放锁后，线程A竞争得到锁继续从wait()方法后执行。

## 参考资料

1. 《java并发编程的艺术》
2. [【Java并发编程】之十：使用wait/notify/notifyAll实现线程间通信的几点重要说明](http://blog.csdn.net/ns_code/article/details/17225469)
3. [Java 并发编程：线程间的协作(wait/notify/sleep/yield/join)](http://www.cnblogs.com/paddix/p/5381958.html)


