# Automic

atomic类原理也没啥好说的，CAS+循环。但是java8中的循环藏起来了，在java8中很多都看不到循环的逻辑了，要跟到openjdk中看Unsafe类的实现，循环基本都藏在那里，当然native方法的除外。
![image.png](/images/atomic1.png)


atomic java8中新的比较有意思的类基本也就是LongAdder了，由于cas一直循环的话会浪费cpu，所以有种更好的办法叫分治法，在源码中也有很多用这种思想的地方，如ForkJoinPool、如这里、还有ConcurrentHashMap的size方法。

基本思想是将一个值分片，不同线程进来后更新不同片区的值。然后sum方法将这些值全部加起来。这样对线程的效率就提高了。
