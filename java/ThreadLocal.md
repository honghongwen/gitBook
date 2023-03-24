# ThreadLocal


### 四种场景


1. 资源持有
每次用户访问接口时，讲用户数据存储在ThreadLocal中，然后用到的时候直接从其中读取。

2. 资源一致性
JDBC中假如一个事务有多个part，每个part都会获取jdbcConnection、但是获取的connection都是一致的，他用threadLocal帮忙存储了，如果是同一个线程，则返回同一个connection，避免事务id出现问题。

3. 线程安全
共享资源，很难去做线程安全，比如常量stat，每次访问时加一，但是多并发的情景下，stat++这个操作不是原子操作，会读取到寄存器中在临界区进行计算，并发场景下多个线程读取的值可能是一致的，后续设置值会覆盖掉老的值

4. 分布式计算
经典的分布式计算模型，divider讲一个大的任务切分为多个小任务，多个线程同时去跑，最后collector收集起来。省时。


### 线程不安全压测

#### 1. 经典++问题
```java
@RestController
public class StatController {

    public static Integer count = 0;
    
    @GetMapping("stat/add")
    public String add() throws InterruptedException {
        count++;
        return "ok";
    }

    @GetMapping("stat")
    public Integer stat() {
        return count;
    }
}

```
> 由于++操作是非原子性的，线程不安全。所以在并发场景下，最终统计的值会有问题。ab（ApacheBench)压测如下。

```shell
ab -n 10000 -c 1 localhost:8080/stat/add
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/454950/1664244579045-874aae79-dfe5-4dd6-8739-ba0de70b14a5.png#averageHue=%23050302&clientId=u9738b210-01d4-4&from=paste&height=623&id=u44ccf82b&name=image.png&originHeight=623&originWidth=700&originalType=binary&ratio=1&rotation=0&showTitle=false&size=34003&status=done&style=none&taskId=udac7e834-7f47-4dd0-899e-24f565ac412&title=&width=700)
```shell
curl localhost:8080/stat -i
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/454950/1664244630653-1f1100f6-c587-48e3-8610-0c5cb585a1cb.png#averageHue=%23090706&clientId=u9738b210-01d4-4&from=paste&height=155&id=u8ebe8719&name=image.png&originHeight=155&originWidth=662&originalType=binary&ratio=1&rotation=0&showTitle=false&size=10336&status=done&style=none&taskId=u0c29ff28-bdcc-434b-b8a3-3edf5278968&title=&width=662)

##### ！可以看到并发为1的情景下结果是无问题的

> 但是提高并发测下

```shell
ab -n 10000 -c 1 localhost:8080/stat/add
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/454950/1664244722203-7dac1fe2-6be0-4616-bf5c-71a90cc29401.png#averageHue=%23060403&clientId=u9738b210-01d4-4&from=paste&height=641&id=u64fab532&name=image.png&originHeight=641&originWidth=637&originalType=binary&ratio=1&rotation=0&showTitle=false&size=34790&status=done&style=none&taskId=uc1de45ea-f2d8-4d4f-a2cc-44ea1908334&title=&width=637)
> 可以看到，在10个并发压测10000个请求。总耗时几乎快了一倍

![image.png](https://cdn.nlark.com/yuque/0/2022/png/454950/1664244792490-690de295-f87a-4f64-b840-e5c512a6fcd0.png#averageHue=%230c0b0a&clientId=u9738b210-01d4-4&from=paste&height=205&id=u9eaa69eb&name=image.png&originHeight=205&originWidth=796&originalType=binary&ratio=1&rotation=0&showTitle=false&size=13340&status=done&style=none&taskId=uc6a3ae60-d323-45b3-8aa4-9d5907d7962&title=&width=796)
##### 但是结果是错误的


#### 2. 加锁
> 第一个想到的肯定是加悲观锁。这样直接就能解决结果不一致的问题。


![image.png](https://cdn.nlark.com/yuque/0/2022/png/454950/1664249651839-06e4426b-ac08-4703-b580-9ba1768106c3.png#averageHue=%23070605&clientId=u9738b210-01d4-4&from=paste&height=683&id=u603d7a63&name=image.png&originHeight=683&originWidth=705&originalType=binary&ratio=1&rotation=0&showTitle=false&size=38177&status=done&style=none&taskId=uecf163fb-9432-4c36-a7c9-edbf21a6f6f&title=&width=705)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/454950/1664249666069-b1d0f781-c2cd-4f6e-98be-3771deb8fc61.png#averageHue=%230d0c0b&clientId=u9738b210-01d4-4&from=paste&height=199&id=ud3c33318&name=image.png&originHeight=199&originWidth=789&originalType=binary&ratio=1&rotation=0&showTitle=false&size=11843&status=done&style=none&taskId=ubb654892-70e2-4cc3-9765-d80ed9db492&title=&width=789)
> 可以看到，解决了数据的正确性问题，但是带来的新问题是，锁是非常耗时的操作，并且在数据量越大的情况下越耗时。所以锁其实是个很危险的操作，不要轻易的去加锁。吞吐量出问题，队列挂掉（队列满了）。


#### 3. ThreadLocal分布式计算
```java
public class StatController {

    static Set<Val<Integer>> set = new HashSet<>();

    static ThreadLocal<Val<Integer>> count = new ThreadLocal<Val<Integer>>() {
        @Override
        protected Val<Integer> initialValue() {
            Val<Integer> val = new Val<>();
            val.set(0);
            addSet(val);
            return val;
        }
    };

    static synchronized void addSet(Val<Integer> val) {
        set.add(val);
    }


    @GetMapping("stat/add")
    public String add() throws InterruptedException {
        Val<Integer> val = count.get();
        val.set(val.get() + 1);
        return "ok";
    }

    @GetMapping("stat")
    public Integer stat() {
       return set.stream().map(x -> x.get()).reduce((sum, item) -> sum + item).get();
    }
}


public class Val<T> {

    T t;

    public void set(T t) {
        this.t = t;
    }

    public T get() {
        return t;
    }
}


```
> 这个计算模型的优点在于，每个线程单独统计其中的值，但是，加锁的数量只有线程的数量（每个线程初始化的时候加一次锁，将val存至set中。方便最后能流计算取出总的值。将锁的数量有每此操作+1变成了常量级可控的数量）   理论上来讲，是有道理的。

### 源码

1. 释放内存
假设是map<Thread, Object> 模型，在线程退出时，自己实现的API无法自动回收内存。要自己判断线程是否退出并且清空hashmap的值。但是Thread内存储了个ThreadLocalMap，这是ThreadLocal自己实现的hashmap。这样在线程exit的时候，直接清空hashmap。至于为什么是hashmap。个人感觉是一个线程中，可能要有多个threadlocal对象。所以干脆用threadlocal做key，然后object做value这样清空。并且其中用到了WeakRefrence<ThreadLcoal>作为Entry的父类。

2. 自己实现的hashmap
应该是基于多方考虑，最终选定使用hashmap作为数据结构，不然的话假如spring的一个线程，里面大家都要线程隔离。那么包括框架方也用threadlocal，然后业务员也用threadlocal。线程Thread类中，如果只是个Object。那无法将对象隔离开。同时，也不好区分key