# Condition

wait和notify线程的等待和唤醒通常是这两个操作，但是这两个操作一般都是和锁挂钩的，而Object里的wait和notify一般是结合sychronized这个锁来使用，不然会抛出异常。之所以会这样，是因为wait的时候会释放掉monitor锁，所以必须结合原生锁来使用。也是为了解决一个lost wake up的问题场景。
而Reentrantlock作为API自己实现的锁，也搞了个线程等待的方法，那就是condition接口。
[condition的解释](https://zhuanlan.zhihu.com/p/97292945)
