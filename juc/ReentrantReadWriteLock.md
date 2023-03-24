# ReentrantReadWriteLock

### ReentrantReadWriteLock读写锁

> 定义：假如有一个资源很少去修改它，但是经常去访问它。此时就可以考虑使用读写锁，因为读写锁能保证多个线程同时读，允许同时读-读。但不允许同时写-读、同时写-写操作。




> 看命名和内部类就知道它同时也能保证公平｜非公平锁，读｜写锁，同时又是可重入锁。


> 用法：模拟一个场景，1000个循环下，获取一个随机数，是偶数则写数据，是奇数则读数据。


```java
public class RwTest {

    public static volatile int salary;

    ReadWriteLock rwLock = new ReentrantReadWriteLock();

    public static void main(String[] args) {
        RwTest rwTest = new RwTest();
        for (int i = 0; i < 1000; i++) {
            int random = new Random().nextInt();
            if ((random & 1) == 0) {
                new Thread(() -> {
                    int salary = 0;
                    try {
                        salary = rwTest.getSalary();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(Thread.currentThread().getName() + " : " + salary + " : " + LocalDateTime.now().format(DateTimeFormatter.ISO_TIME));
                }, "Read-" + i + "-" + random).start();
            } else {
                new Thread(() -> {
                    int salary = 0;
                    try {
                        salary = rwTest.setSalary();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(Thread.currentThread().getName() + " : " + salary + " : " + LocalDateTime.now().format(DateTimeFormatter.ISO_TIME));
                }, "Write-" + i + "-" + random).start();
            }
        }
    }


    public int getSalary() throws InterruptedException {
        Lock readLock = rwLock.readLock();
        readLock.lock();
        try {
            Thread.sleep(1000);
            return salary;
        } finally {
            readLock.unlock();
        }
    }

    public int setSalary() throws InterruptedException {
        Lock writeLock = rwLock.writeLock();
        writeLock.lock();
        try {
            Thread.sleep(1000);
            salary += 100;
            return salary;
        } finally {
            writeLock.unlock();
        }
    }
}
```

> 运行结果如下，通过时间可以看出，读锁和写锁之间是互斥的，写锁和写锁之间也是互斥的，但是读锁和读锁之间是共享的。




> 关于源码的阅读方面，读写锁的实现里有几个需要先了解的概念。
>  
> 1. 原先的AQS中state被拆分为了高16位和低16位，共享锁针对高16位取值，而排他锁针对低16位取值。
> 2. 由于读锁是所有读线程都可以获取的，同时又是可重入的，所以搞了个ThreadLocal存放每个线程获取读锁的个数。
> 
 


了解完这些概念，再去读源码。读源码肯定是跟着功能接口往下走。

首先针对读锁，加锁的源码如下。

```java
        public void lock() {
            sync.acquireShared(1);
        }
        
        // AQS模版
        public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
        }
        
        protected final int tryAcquireShared(int unused) {
            
            Thread current = Thread.currentThread();
            // 熟悉的state
            int c = getState();
            // 如果有写锁并且AOS中排他线程不是当前线程，直接失败返回等待
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            // 如果读锁不用阻塞且读锁小于最大数量且CAS成功 readerShouldBlock在FairSync和NonfairSync中有不同实现   
            int r = sharedCount(c);
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                // 走到这一步表明CAS成功了，这里就是在存Threadlocal中锁的数量了。
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            // 如果读锁要等待或者CAS失败了
            return fullTryAcquireShared(current);
        }
```

```java
final int fullTryAcquireShared(Thread current) {
            /*
             * This code is in part redundant with that in
             * tryAcquireShared but is simpler overall by not
             * complicating tryAcquireShared with interactions between
             * retries and lazily reading hold counts.
             */
            HoldCounter rh = null;
            for (;;) {
                // 同之前判断写锁并且排他线程
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                    // 如果当前线程之前就获取过锁，代表是在重入，此时不为写锁让步，否则看下面
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        // 如果之前没有获取过锁，则不是重入，此时为写锁让步，避免写锁饥饿。
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // CAS操作    
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }
```

> 其实翻译过来。读锁的加锁逻辑十分清晰，只不过state被分为高16位和低16位以及存储线程中锁的数量的实现比较复杂。


> 再看写锁的加锁逻辑。其实也十分简单。搞清楚怎么判断读锁或写锁的位运算后，加锁的这部分代码就非常清晰。同样写锁是否需要等待也分公平锁和非公平锁的实现，非公平锁直接就是所有都false.公平锁则需判断头节点下一个节点是不是自己的逻辑。


```java
    // 同样走到熟悉的AQS模版方法。看读写锁里Sync的具体实现
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    
    protected final boolean tryAcquire(int acquires) {
            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c);
            // 如果有锁
            if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                // 如果有写锁，并且排他线程不是当前线程，直接入队列等待
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;    
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                // CAS操作
                setState(c + acquires);
                return true;
            }
            // 如果写锁需要等待或者CAS失败，入队列等待。
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            // 设置排他线程    
            setExclusiveOwnerThread(current);
            return true;
        }
```

关于读写锁源码的阅读，这篇文章写的非常好，同时读写锁的实现也不复杂。[文章地址](https://juejin.cn/post/6844903624338849805) 同时也能更好的理解为嘛有了AQS中的state状态，还要搞个AOS的exclusiveThread属性。每次CAS成功后设置此值。这个其实就是个排他锁的概念，同时也是可重入锁的重要依赖。每次判断state != 0，证明其他线程正在持有锁，此时判断持有锁的线程是不是当前线程，如果是的话就能再次获取锁，实现重入。
