# ArrayList

### 1.扩容

> ArrayList搞了个minCapacity（最小容量）的概念，抽出了个方法。参数就是minCapacity.


定义如下：
`void ensureCapacityInternal(int minCapacity);`

> 每次向list内加元素的时候，预先设置好需要的最小容量，传入ensureCapacityInternal中，进行后续是否扩容操作。

后续代码就较为简单，单独方法calculateCapacity判断list是不是默认的空参初始化，list无容量时从default_capacity和minCapacity取较大那个。
再后续时就是具体的扩容机制，确保有足够的容量装配元素。
`ensureExplicitCapacity(int minCapacity)`
当minCapacity大于当前的elements.length时，就需要进行扩容操作了。
代码较为简单：

具体的扩容机制就在grow方法内。
>  
> 1. 新容量为老容量+老容量*0.5。
> 2. 如果新容量还是小于最小容量，新容量直接改为最小容量。
> 3. 大容量的操作（一般不考虑。）
> 
 


扩容具体用到的数组复制方法就是Arrays.copyOf了，底层就是调用的System.arraycopy.

> !大容量作考虑是因为影响list扩容的永远是内存不够，而不会是容量到达上限。
int的值上限大于20亿。而一般情况下一个List如果有100w条数据的话。
那就是100w*4byte=400w个字节。4000000/1024/1024=3.8M
而一般的对象肯定不止4个字节，假如将这个数据扩大到32byte（对象头+padding信息，对象很容易就达到这个数值。）那100w的数据很容易达到几十上百M，而元素再扩大到100w以上后，OOM就是很容易触发的事情了。


其实ArrayList里的minCapacity这个概念是非常好的，以前自己实现一个能自动扩容的集合。没有设计的这么复杂，只是每次装不下后将其容量扩大到两倍这样子。并未考虑到批量添加时的扩容情形。

```java
public class List<E> implements IList<E> {

    private Object[] elements; // 数组
    private int index = -1; // 下标
    private final static int DEFAULT_SIZE = 10; // 数组默认长度

    public List() {
        elements = new Object[DEFAULT_SIZE];
    }

    @Override
    public void add(E e) {
        // 扩容
        if (elements.length == index + 1) {
            addScale();
        }
        // 赋值
        elements[++index] = e;
    }

    private void addScale() {
        // 扩到原数组两倍
        int size = elements.length;
        size = size * 2;
        Object[] temp = new Object[size];
        System.arraycopy(elements, 0, temp, 0, elements.length);
        elements = temp;
    }

    @Override
    public int remove(E e) {
        for (int i = 0; i <= index; i++) {
            Object o = elements[i];
            if (o == e) {
                int numToMove = elements.length - 1 - i;
                System.arraycopy(elements, i+1, elements, i, numToMove);
                elements[elements.length - 1] = null;
                return i;
            }
        }
        return -1;
    }

    @Override
    public void set(int index, E e) {
        elements[index] = e;
    }

    @SuppressWarnings("unchecked")
    @Override
    public E get(int index) {
        return (E) elements[index];
    }

    @Override
    public int size() {
        return index + 1;
    }
}
```
