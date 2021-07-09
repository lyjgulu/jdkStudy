# ReentrantLock

## 非公平锁

**state使用volatile关键字修饰保证可见性**

``` java
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))	// 第一次尝试获取锁
                setExclusiveOwnerThread(Thread.currentThread());	// 已成功获取锁，设置持有锁的线程为当前线程
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```

### acquire(1)

``` java
    public final void acquire(int arg) {
      	// tryAcquire 有四种实现分别对应公平锁、非公平锁、读写锁、线程池，注意这里是 （!），意味获取锁成功就不会执行 (&&) 后面的代码
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

### tryAcquire(arg)

``` java
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
```

### nonfairTryAcquire(acquires)

``` java
        /**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();	// 锁的状态，可以简单理解为锁的次数
            if (c == 0) {	// 锁没有被获取
                if (compareAndSetState(0, acquires)) {	// cas获取锁，第二次获取锁
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {	// 这里是重入锁部分
                int nextc = c + acquires;
                if (nextc < 0) // overflow 超过了最大重入次数
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

**前两次获取锁都失败，执行入队操作**

### addWaiter(Node.EXCLUSIVE)    EXCLUSIVE 表示节点正在以独占模式等待的标记

``` java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);	// 新建一个当前线程且为独占模式的 Node 
      	// 因为传入的mode值为Node.EXCLUSIVE，所以节点的 nextWaiter 属性被设为 null
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
      	// 队列是采用延迟初始化操作的，如果队列不为空，就使用 CAS 操作将当前节点设置为尾节点
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
              	// 注意！node 节点的前一个节点在设置完 node 为尾节点后才会将 pred 的 next 指针指向 node，中间会有一个时间差
                pred.next = node;
                return node;
            }
        }
        // 代码会执行到这里, 只有两种情况:
        //    1. 队列为空
        //    2. CAS失败
        // 注意, 这里是并发条件下, 所以什么都有可能发生, 尤其注意CAS失败后也会来到这里
        enq(node);
        return node;
    }
```

每一个处于独占锁模式下的节点，它的 nextWaiter 一定是 null。
在这个方法中，我们首先会尝试直接入队，但是因为目前是在并发条件下，所以有可能同一时刻，有多个线程都在尝试入队，导致 compareAndSetTail(pred, node) 操作失败——因为有可能其他线程已经成为了新的尾节点，导致尾节点不再是我们之前看到的那个 pred 了。

### enq(node)

``` java
    /**
     * Inserts node into queue, initializing if necessary. See picture above.
     * @param node the node to insert
     * @return node's predecessor
     */
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
          	// 空队列进行初始化
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node())) // 将 head 指向新的空节点
                    tail = head;	// 将 tail 也指向新的空节点
            } else {
              	// 队列不是空的，继续尝试加入队尾巴
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

**这里入队会出现比较特殊的情况，主要是并发操作的原因**

添加node节点到尾部的三个步骤：

1. 设置 node 的前驱节点为当前的尾节点：`node.prev = t`
2. 修改 tail 属性，使它指向当前节点
3. 修改原来的尾节点，使它的 next 指向当前节点

![enqStep](https://raw.githubusercontent.com/lyjgulu/jdkStudy/main/ReentrantLock/image/enqStep.png)

但是，这里的三步并不是一个原子操作，第一步很容易成功；而第二步由于是一个 CAS 操作，在并发条件下有可能失败，第三步只有在第二步成功的条件下才执行。这里的 CAS 保证了同一时刻只有一个节点能成为尾节点，其他节点将失败，失败后将回到 for 循环中继续重试。

所以，当有大量的线程在同时入队的时候，同一时刻，只有一个线程能完整地完成这三步，而其他线程只能完成第一步，于是就出现了尾分叉：

![enqPhenomenon](https://raw.githubusercontent.com/lyjgulu/jdkStudy/main/ReentrantLock/image/enqPhenomenon.png)

会出现`prev.next = null`的情况，因此在 AQS 源码中常见的都是从尾部向前遍历

### acquireQueued(final Node node, int arg)

``` java
      /**
       * Acquires in exclusive uninterruptible mode for thread already in
       * queue. Used by condition wait methods as well as acquire.
       *
       * @param node the node
       * @param arg the acquire argument
       * @return {@code true} if interrupted while waiting
       */
			final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();	// p 为 node 节点的前一个节点
                if (p == head && tryAcquire(arg)) {	// 如果 p 是 head 节点，再次尝试获取锁，这已经是最起码三次以上尝试获取锁
                    setHead(node);	// 设置 node 节点为新的头结点
                  	/*private void setHead(Node node) {
        							head = node;					// 非常重要，头结点是近似一个空节点
        							node.thread = null;
        							node.prev = null;
    								}*/
                  
    								p.next = null;	// help GC	JVM 采用 GC Root 的方式寻找可回收的垃圾
                    failed = false;
                    return interrupted;
                }
              	// 获取失败是否需要把当前线程挂起(park)
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
              	// 应用于可中转抢占锁
                cancelAcquire(node);
        }
    }
```

### setHead(node)

![enqStep](https://raw.githubusercontent.com/lyjgulu/jdkStudy/main/ReentrantLock/image/enqStep.png)

### 四种 waitStatus

**非常重要，node 的 waitStatus 存储在前一个 node 中**

``` java
				/** waitStatus value to indicate thread has cancelled */
				// 指示线程已取消
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
				// 指示 next 线程需要 park
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
				// 指示线程正在等待条件
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
				// 指示下一个acquireShared 应无条件传播
        static final int PROPAGATE = -3;
```

### shouldParkAfterFailedAcquire(p, node)

``` java
    /**
     * Checks and updates status for a node that failed to acquire.
     * Returns true if thread should block. This is the main signal
     * control in all acquire loops.  Requires that pred == node.prev.
     *
     * @param pred node's predecessor holding status
     * @param node the node
     * @return {@code true} if thread should block
     */
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
          	// 指示该 node 进入睡眠状态
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
          	// 当前节点的 ws > 0, 则为 Node.CANCELLED 说明前驱节点已经取消了等待锁(由于超时或者中断等原因)
            // 既然前驱节点不等了, 那就继续往前找, 直到找到一个还在等待锁的节点
            // 然后我们跨过这些不等待锁的节点, 直接排在等待锁的节点的后面
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
          	// 前驱节点的状态既不是 SIGNAL，也不是 CANCELLED，用 CAS 设置前驱节点的 ws 为 Node.SIGNAL
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

### parkAndCheckInterrupt()

``` java
    /**
     * Convenience method to park and then check if interrupted
     *
     * @return {@code true} if interrupted
     */
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);	// 线程被挂起，处于这个状态只能由其他线程 unpark 唤醒或者是被中断
        return Thread.interrupted();
    }
```

**需要前一个节点释放锁的时候对这个 node 进行唤醒**

## 释放锁

### release(int arg)

``` java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

### tryRelease(arg)

``` java
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {	// 判断是否当前锁已经完全释放
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

### unparkSuccessor(h)

``` java
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
          	// 小于 0，直接设置为0
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
      	// 正常是唤醒后继节点。
        Node s = node.next;
      	// 没有后继节点或者是后继节点取消了等待锁
        if (s == null || s.waitStatus > 0) {
            s = null;
          	// 此时从尾节点向前遍历找到距离头节点最近的正在等待获取锁的节点
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
          	// 找到了就去唤醒，非常重要！
            LockSupport.unpark(s.thread);
    }
```

