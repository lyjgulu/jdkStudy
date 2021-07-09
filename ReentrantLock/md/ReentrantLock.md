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



