# Semaphore

> 可以理解为一个限流器。个人觉得他能实现的东西，Condition接口也能实现，不过condition接口的实现没如此灵活。


> 使用方法如下：


```java

public class SemaphoreTest {

    Semaphore semaphore = new Semaphore(2);

    public static void main(String[] args) {
        SemaphoreTest semaphoreTest = new SemaphoreTest();
        for (int i = 0; i < 20; i++) {
            new Thread(() -> {
                try {
                    semaphoreTest.enterRoom();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }, "Thread-" + i).start();
        }
    }

    public void enterRoom() throws InterruptedException {
        if (semaphore.availablePermits() == 0) {
            System.out.println(Thread.currentThread().getName() + " : " + "被禁止访问了...");
        }

        semaphore.acquire();
        System.out.println(Thread.currentThread().getName() + " : " + "成功进入房间，开始嗨皮。");
        Thread.sleep(1000);
        System.out.println(Thread.currentThread().getName() + " : " + "离开了房间。");
        semaphore.release();
    }
}
```

![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1632642478838-cb8d9825-d7c9-4bf3-a73c-a7bf85fcbb16.png#clientId=ue1929891-386e-4&from=paste&height=931&id=u7778fb83&name=image.png&originHeight=931&originWidth=1814&originalType=binary&ratio=1&size=166678&status=done&style=none&taskId=u72442b7c-6827-41c8-bf05-363fb49109f&width=1814)
