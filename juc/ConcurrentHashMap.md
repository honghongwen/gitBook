# ConcurrentHashMap

### 1.如何保证线程安全

线程安全的概念：

> 与其说是线程安全，不如说是内存共享区域的数据安全。因为在内存中，堆中数据是所有线程共享的。共同可操作的结果就导致其私密性不好。由此可见有下面几种办法处理：


1.不使用堆空间保存私密数据

> 如果要保证私密性，则将要用到的数据作为方法的局部变量，这样由于局部变量是分配在方法栈中，只能被此线程访问。所以能够保证数据的私密性。


2.使用ThreadLocal

> 线程实例中有一个变量用于专门保存线程私密的数据。就好像是每个线程都认领了一份专属的数据去操作，虽然从本质上来说数据的位置缺失还是在内存共享区域中，但由于每个线程都有自己独有的一份，所以也能保证不受其他线程的干扰。


3.使用final修饰

> 使用final修饰的成员变量只能被读取，但是不能被修改，自然是线程安全的。


4.加锁

> 使用synchronized或者lock等对要访问的资源加锁，每次想要操作数据的时候，都必须持有锁，而无锁的线程将无法对资源进行操作（互斥锁）。


5.相信世界充满爱CAS

> 加锁和释放锁是需要一定代价的，如果线程数目特别少的情况下，可能根本就不会有别的线程来操作数据，此时加锁就是一种浪费。而CAS指的是我假定数据不会被人修改，但以防万一每次线程切换上下文时我还会校验下数据是不是和之前的一致，如果一致，那就无问题。如果不一致，那就放弃重新操作。


> ps:一致的情况下还有ABA的问题，为了防止这种情况，可以加个字段版本号version.


### 2.ConcurrentHashMap中的线程安全

#### 1.CAS + 线程自旋

> ConcurrentHashMap中用了大量的CAS，同时还用到了synchronized锁。比如在initTable初始化数组方法中。
首先当table==null的情况下进入循环体，之后通过CAS，导致只有一个线程能成功的修改sizeCtl标识。其他线程修改失败后由于sizeCtl被改为-1，直接重新等cpu调度去了。也就是旋转的实现。而CAS成功的线程初始化table完成后，才在finally块中将sizeCtl修改为正数。旋转中的线程才可以继续进入代码块，但此时由于table已被初始化，直接break出循环结束了操作。后续线程亦如此。




#### 2.头节点锁synchronized(f) 取代了jdk7的分段锁。

> concurrentHashMap中一个比较有名的优化就是分段锁了，每次putVal的时候，出于对性能的考虑，没用sychronized关键字对整个方法加锁，而是对数组下标中链表的首个节点进行加锁，这样加入后续元素被put进来时会有一定性能上的提升，而且把treeify操作放在了锁外。


#### 3.扩容

> concurrentHashMap最难啃的一块我认为就是扩容操作了。因为他是一个并发操作，允许多线程操作。putVal方法中就有如果f.hash == MOVED就helpTransfer的代码。而MOVED是只有Node变成ForwardingNode的时候，才会变为-1，而只有在扩容的时候，Node才会变为ForwardingNode。也就是说，如果有线程在进行扩容操作，那其他线程在putVal的时候，也会参与到协助扩容到操作里来。（毕竟如果不是这样，那也会被锁住浪费资源。大牛的考虑）。接下来就是最关键的知识点，扩容代码transfer了。大致从这段代码的意思是每个线程领取一段数组（最小16）然后进行数据转移。具体代码如下：


```java
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
        }
        int nextn = nextTab.length;
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```
