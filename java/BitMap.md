# BitMap

### 判断大数据量集合中存不存在某一个元素。
> 假设有10亿个数字的集合，要判断这个某个数字在不在改集合中。如何处理。

> 在java中int为32个位4个字节，10亿个就是40亿个字节，换算后大概需要4个g的内存。
> 所有有一个办法是让每个字节保存8个数，如下

> 8在bitmap中就如下 第8位为1就代表8在集合中。

> 0000 0001


> 那按照这个规律，一个int可以表示32个数在不在集合中，那表示10亿个数字不在字节中，所需的内存大大减小。大约只需要120m的内存就可以表示了。


那按这个方案，伪代码就可以写出来了。
```java
public class BitMap{
  // 数组越长，hash碰撞几率就越低。效率越好。但内存越高
  int[] arr = new int[35000000];
    
  public void add(int i) {
  	int aIndex = i / 32;
    int nIndex = i % 32;
    // 数字落到aIndex这个下标处并且该下标的数  1左移nIndex位 
    arr[aIndex] = nums[aIndex] | 1 << nIndex;
  }
  
  public boolean contails(int i) {
  	int aIndex = i / 32;
    int nIndex = i % 32;
    // 其他位清零，只取nIndex这个位置数字
    return ((arr[aIndex] >>> nIndex) & 1) == 1 
  }  
} 
```

基于这个方案，还有个升级版 布隆过滤器。
> 如上所示，判断字符串并不是非常方便，当然，你也可以套层hash。但是hash冲突需要解决好。

> 在这个基础上，升级的布隆过滤器，通过定义了几个过滤器，分别拥有不同的hash方法，
> 那一个字符串进来后，会每个过滤器中hash一次并将结果放入bitmap中。
> 同样，判断是否在集合中时，也是判断bitmap中是否有每个字符的hash值。

