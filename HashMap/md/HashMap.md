# HashMap

- 一些重要的全局常量和方法

``` java
    /**
     * The default initial capacity - MUST be a power of two.默认初始容量 - 必须是 2 的幂。(hash桶数量)
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * The load factor used when none specified in constructor.空间与时间的平衡，减少 resize 操作，与泊松分布没有直接关系，理想随机hash算法满足泊松分布。
     * 一个 bucket(桶) 空与非空概率为 0.5，通过牛顿二项式等数学计算，得到精准 loadfactor 的值为ln2 ~ 0.693，本人猜想为了符合bucket是 2 的幂，计算出的扩容阈值为正数，故使扩容因子为 0.75。
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2 and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.			链表长度大于 8 转换红黑树，转换成红黑树条件一。
     * 理想随机 hash 出现 hash 冲突拉出长为 8 的链表概率为百万分之一。
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * The bin count threshold for untreeifying a (split) bin during a
     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
     * most 6 to mesh with shrinkage detection under removal.
     * 树高小于 6，转换成链表
     */
    static final int UNTREEIFY_THRESHOLD = 6;

		/**
     * The smallest table capacity for which bins may be treeified.
     * (Otherwise the table is resized if too many nodes in a bin.)
     * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
     * between resizing and treeification thresholds.’
     * 满足条件一之后，还需要满足 bucket(桶)大于 64，此为转换红黑树条件二，通过 resize 操作来满足
     */
    static final int MIN_TREEIFY_CAPACITY = 64;

				// hashCode 的计算方法。( ^ 如果相对应位值相同，则结果为0，否则为1)( ^ 异或)
				public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

		// hash 值的计算方法
		static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

- `get`

``` java
		public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
		
		final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
      	// 判断当前 hashMap 是否为空。
        if ((tab = table) != null && (n = tab.length) > 0 &&
            // first = bucket(桶)位置的值，也就是数组下标当前位置的 Node 节点。
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
              	// 没有理解为什么还要 key != null && key.equals(k) 这个条件。
              	// hash 值相同并且 key 相等直接返回 first 这个 Node。
                return first;
            if ((e = first.next) != null) {
            // 上面条件不符合，但 first 这个 Node 又不为空，且 first 的下一个 Node 也不为空，说明这个 bucket(桶)位置形成链表或红黑树。
                if (first instanceof TreeNode)
                  	// 是树节点，返回树节点符合条件的 Node。
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
              	// 遍历链表取 Node。
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
      	// 找不到返回空。
        return null;
    }
```

- `put`

``` java
		public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

		/**
     * Implements Map.put and related methods.
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
          	// hashMap 判空决定是否要扩容，实际上是初始化。
          	// 将数组长度赋值给 n。
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
          	// 当前桶位置为空，直接在这个桶位置创建一个新的 Node。 
            tab[i] = newNode(hash, key, value, null);
        else {
          	// 该桶位置已经有 Node，可能需要进行链表或红黑树操作。
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
              	// 覆盖当前桶位置的第一个值的 value。
                e = p;
            else if (p instanceof TreeNode)
              	// 已经是红黑树。
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                      	// 尾插法
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                          // 判断链表的长度是否需要转换红黑树
                          treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                      	// 链表中找到相同 key 的节点
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
              	// 根据参数判断是否要返回该节点的不为空的旧值
                return oldValue;
            }
        }
      	// 操作次数
        ++modCount;
      	// 判断是否需要扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

- `resize`

``` java
		/**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     * 初始化或加倍表大小。如果为空，则根据字段阈值中持有的初始容量目标进行分配。
     * 否则，因为我们使用的是 2 的幂扩展，所以每个 bin 中的元素必须保持相同的索引，或者在新表中以 2 的幂的偏移量移动。
     * @return the table
     */
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
      	// 旧的数组长度
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
      	// 旧的扩容阈值
        int oldThr = threshold;
      	// 新的数组长度，新的扩容阈值
        int newCap, newThr = 0;
        if (oldCap > 0) {
          	// 超出最大值
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
              	// 新的扩容阈值是旧的扩容阈值
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
      	// 计算新的resize
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                      	// 没有链表和红黑树找到新的桶位置
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                      	// 红黑树
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                      	// 链表
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                          	// 原来的桶位置
                          	// 如果该节点的(e.hash & oldCap) == 0，说明(e.hash & newCap - 1) == j
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                          	// 原来的桶位置加上原来 hashMap 的长度
                          	// 否则就是(e.hash & newCap - 1) =j + oldtab
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                      	// 原理桶位置放到 bucket 中
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                     		// 原来的桶位置加上原来 hashMap 的长度放到新的 bucket 中
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

