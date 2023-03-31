# Exchanger

> 很有意思的一个小工具，能直接等到两个线程都执行到exchange中然后交换数据。


```java
public class UseExchanger {

    public static void main(String[] args) {
        Exchanger<Human> exchanger = new Exchanger<>();

        new Thread(() -> {
            try {
                Thread.sleep(10000);
                Human feng = new Human("feng wen", 26, "男");
                Human changed = exchanger.exchange(feng);
                System.out.println(Thread.currentThread().getName() + ":" + changed);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "Thread-Feng").start();


        new Thread(() -> {
            try {
                Thread.sleep(2000);
                Human yi = new Human("lin yi", 25, "女");
                Human changed = exchanger.exchange(yi);
                System.out.println(Thread.currentThread().getName() + ":" + changed);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "Thread-Yi").start();

    }
}


class Human {

    private String name;

    private int age;

    private String gender;

    public Human(String name, int age, String gender) {
        this.name = name;
        this.age = age;
        this.gender = gender;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getGender() {
        return gender;
    }

    public void setGender(String gender) {
        this.gender = gender;
    }

    @Override
    public String toString() {
        return "Human{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", gender='" + gender + '\'' +
                '}';
    }
}
```



##### 实现原理：
