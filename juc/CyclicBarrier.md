# CyclicBarrier

> 这个可是模拟并发的神奇，同样是juc中的小工具，和CountDownLatch相反的是它要所有线程await.直到所有线程都到了await点到时候才会一起继续执行。


> 理解起来也很简单。[示例](https://www.jianshu.com/p/4ef4bbf01811)


> 源码：
它的实现也很简单，构造函数初始化了个count值。其他线程中每次await方法会用到lock。
同时也会用到condition。每次await的时候加锁，然后--count。之后condition.await()让此线程等待。 直到触发count==0这个阈值后，说明所有线程都在await了。这个时候先查看是否有钩子线程，如果有，先执行它（保存了对应的runnable然后调用run方法）。再之后唤醒所有线程。也就是调用condition.signalAll方法。

