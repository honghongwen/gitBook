# CountDownLatch

> countDownLatch使用的时候感觉还挺神奇的，但是知道原理后实在没什么好说的。就是用了AQS里的state状态，其他线程调用countDown去CAS --state。而await的线程会将state设为构造参数中传入的值。太简单了。

