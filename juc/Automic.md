# Automic

atomic类原理也没啥好说的，CAS+循环。但是java8中的循环藏起来了，在java8中很多都看不到循环的逻辑了，要跟到openjdk中看Unsafe类的实现，循环基本都藏在那里，当然native方法的除外。
![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1632997502480-4dbbd2e4-aa29-4f0f-870d-f30b73dcb71d.png#clientId=ub507a2c3-636d-4&from=paste&height=927&id=ufcc94b91&name=image.png&originHeight=927&originWidth=1358&originalType=binary&ratio=1&size=164419&status=done&style=none&taskId=u0b1738c6-9162-4937-b2e7-a700b4b226a&width=1358)


atomic java8中新的比较有意思的类基本也就是LongAdder了，由于cas一直循环的话会浪费cpu，所以有种更好的办法叫分治法，在源码中也有很多用这种思想的地方，如ForkJoinPool、如这里、还有ConcurrentHashMap的size方法。

基本思想是将一个值分片，不同线程进来后更新不同片区的值。然后sum方法将这些值全部加起来。这样对线程的效率就提高了。
