---
layout: post
title:  "实现支持消息合并机制的并发阻塞队列"
date:   2015-12-09 10:18:00
permalink: implement-of-a-hash-blockingqueue
---

最近我维护的一个系统（下文简称 A）需要实现一个消息队列，大致的场景是：B系统监控某任务的状态，当状态有变化时通过调用 A 的 RESTful 接口请求将最新的状态推送至 A，A 接受到消息后将其插入到自己的消息队列中，然后再从队列中取出消息并再广播给第三方的系统。针对该场景，该消息队列需要支持以下两个主要特性：

1. 保证消息的接受及发送的 FIFO .
2. 支持消息合并。由于消息的内容是对特定任务不同状态的描述，若该任务有最新消息到达，那么已接受到的该任务之前的消息需要从消息队列中去除而不必再进行推送。

实现消息生产-消费的 FIFO，在 Java 中使用 BlockingQueue 就能很容易的实现，再加上利用线程池来对消息进行消费，能够达到较高的性能。

而要实现消息的合并就会麻烦一点，通常在 Queue 上更新某个元素需要遍历整个队列。而采用直接组合 HashMap + Queue 方式又不能保证线程安全性。

拍脑袋决定直接在 BlockingQueue 源代码的基础上进行改造，通过在更新 Queue 和 HashMap 时使用同一个 lock 由来保证操作的原子性。这样构造出一个 HashBlockingQueue 就基本上满足上述需求了。由于系统中该消息队列需要是 unbound 的，所以采用了 LinkedBlockingQueue(LBQ) 来实现上述功能。

翻了翻 LBQ 的代码，其采用了 putLock 和 takeLock 两个 ReentranLock 来保证添加和删除时的原子性,并且在上述两个 Lock 上分别添加两个条件变量来实现阻塞等待：

```java
    /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();
```

实现阻塞队列的功能主要用到了以下几个方法：

```java
public void put(E e) throws InterruptedException;  
public E take() throws InterruptedException;  
```

LBQ在实现 put 和 take 方法的中的用到了一些提高并发度的技巧：

使用两个 Lock 来分别控制头部删除以及尾部的插入，这样 put 和 take 两个操作就不会相互影响，提高了并发度。
对 count (当前队列元素的数量) 的操作不加锁。以 put 为例，count 变量在临界区进行 count.get()操作，此时可能有另外一个 take 线程将会更新 count 值(decrease), put线程会因为脏读而进入 wait 状态。但是因为 3. 中将会提到的： put 线程只会 wait 在 putLock 上，且 wait 状态的 put 线程可以由其它 put 线程来唤醒(signal)，所以不会出现死锁的问题。
由于 1 的缘故，put 和 take 操作可以分别 wait 在各自 lock 的条件变量上, 例如在 put 方法中：

```java
  /* put */
   final ReentrantLock putLock = this.putLock;
   final AtomicInteger count = this.count;
   putLock.lockInterruptibly();
   try {   
       while (count.get() == capacity) {
           notFull.await();
       }
       enqueue(node);
       c = count.getAndIncrement();
       if (c + 1 < capacity)
           notFull.signal();
   } finally {
       putLock.unlock();
   }
      if (c == 0)
         signalNotEmpty();

         ......
```

这样做好处是在大多数情况下进一步降低了 take 和 put 操作相互之间影响。例如，当 P1 线程调用 put 方法因为容量已满将会阻塞在 notFull.await()，此时其会释放 putLock，从而让其它调用 put 线程有机会进入临界区，以此来不断的尝试进行 put 操作，一旦有一个调用 put 线程成功，其会根据条件 (c + 1 < capacity)尝试唤醒阻塞在 notFull 条件变量上的其它 put 线程。take 方法与上述机制类似，这里就不详述了。

put 和 take 方法中通过调用 enqueue 和 dequeue 来实现队列的添加和删除，因此，我们只需将我们对 HashMap 的操作放入上述两个函数中，就能实现 HashMap 与 Queue 更新的原子性。为了实现消息的合并，我们通过重载 put 方法，传入一个函数对象以实现根据具体的业务逻辑来判断是否需要更新消息队列中消息的状态，伪代码如下：

```java
 public void put(E e, Function fn) throws InterruptedException {
     ...
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            if (hashMap.containsKey(e.getEventName())) {
                Node<E> n = hashMap.get(e.getEventName());
                if (((Boolean) fn.apply(e, n.item))) {
                    n.item.setStatus(EventStatusType.ABORTED);
                }
            }

            // put normally.
            ...
```

上文中 E 为泛型类型，代表队列中消息的类型，继承自一个统一的事件类型接口。从上述代码片段可以看到，如果当前队列包含当前事件，且作为业务逻辑判断条件的函数 fn 返回 true 则将队列中已有的消息标记为 Aborted。此后，当消费线程取到 Aborted 的消息时将自动抛弃该消息对象。

在实现了以上的改造后，该 HashBlockingQueue 支持 O(1) 时间复杂度的头部插入和尾部删除，O(1) 复杂度对队列中的节点进行更新，并且能够保证线程安全性。于是，愉快地将该消息队列上线到生产系统中。

在实现了上述 HashBlockingQueue 后，一段时间内已经能够满足业务需求。但是后来碰到一个新的问题：上文的实现中，对于重复的消息，采用了将过期消息状态标记为无效，并且在消费者线程中来判断消息有效性的机制，但是随着消息量的增长，一方面：系统经常会遇到一个消息的状态频繁更新的状况，此时，队列中将会充斥了大量无效的消息，另一方面：由于在该系统所面对的场景中，消费线程需要通过 http 请求的方式回调第三方系统从而完成消息的消费，消费速度很难提高，大量消息的堆积大大增加了整个系统的内存占用量，严重影响了系统的性能。

为了实现快速的删除无效的消息（而非仅仅进行标记），我将上文中的 HashBlockingQueue 改为由双向链表实现从而支持了 O(1) 时间复杂度的节点删除，这也意味着我们在有事件更新的时候能够快速的删除无效的事件，保证整个队列的长度在可控的范围，具体的实现就不再详细阐述了 :)