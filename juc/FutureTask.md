# FutureTask

> 都知道启动一个线程要调用new Thread(Runnable runnable).start方法。而Runnable就好比是一个任务，真正执行他的是Thread的start方法。线程池里的ExecutorService扩展了提交任务的方式，允许你获取一个Future来获取返回值，而这种Runnable是没有返回值可以获取的，所以有个方法submit(Runnable runnable, T t)，让你将返回值的类型传进来，任务执行后可以通过Future接口get到具体的结果。  
> 
> 但是每次都这么搞好像感觉怪怪的，也不太方便。所以定义了一种能获取返回值的任务，叫Callable<T>接口。

### 分析后总结如下：
> AbstartExecutorService定义了submit方法，允许你提交两种类型的任务，Callable或者Runnable。这两种接口最终都会被封装成一个FutureTask，这个就是最终的任务类。
> 
> 1. 这个task也实现了runnable接口，那最终调用的就是task的run方法。
> 2. 同时FutureTask还依赖了个Callable, task运行run方法的时候，其实里面调用的是Callable的call方法。
> 3. task依赖callable,而如果是runnable类型的任务，会先被适配成callable。 Executors.toCallable(Runnable runnable)完成了这件适配工作。适配后传给task。


### 实现原理
> 实现原理很简单，FutureTask有个状态值，call方法之后才CAS设置状态为COMPLETED，而在之前调用get方法的话，如果state没有到COMPLETED状态的话，就会一直阻塞住。

