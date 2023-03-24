# HashMap

### 1.数据结构

HashMap有个静态内部类Node,它是Map内部接口Entry的实现。它的内部有几个属性。

> 1.int hash
2.K key
3.V value
4.Node next


同时外部有个Node数组，由此可见，他是一个数组+链表的实现方式。对比起单纯的数组格式。他有以下的好处。

> 寻址快，如果是单纯的数组，通过key去进行寻址，最差的情况每次都需要遍历整个数组。那进行元素增加和获取元素的时候，有点太糟糕了。而这种数据结构是一个很棒的设计，每次根据hash来确定数组的索引。然后将相同索引下的元素都放在同一个数组索引下，组成链表的形式。


引申而来的是依据hash进行下标定位时需要有一个较好的hash算法。这样才可能比较均匀的将元素放在不同的数组下标上。不然可能加入一批元素，他们全部放在数组[0]这个位置，这样就成了一个完全的链表了。那和数组也没啥区别。一样造成寻址缓慢。

HashMap中有一个关键方法： putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) 具体如下：



同时它还有个很有名的hash算法。又叫扰动函数：

hash是通过hash(K key)这个方法计算出来的。这个方法又叫扰动函数，是一个很有讲究的算法，具体见[知乎解答](https://www.zhihu.com/question/20733617)。它也解释了为什么hashmap的数组长度总是2的整数次幂。大概的意思就是每次拿（length-1）& hash计算下标的时候。如果单纯是这样计算，length较小的情况下，其实就是在取hash值的最后几位，再加上假如hash本身就做的不够好的情况下，hash呈现有规律的结果，很容易就造成hash碰撞。也就是上述的最糟糕情况下全部在数组[0]这个位置储存元素。而扰动函数中拿hashcode的高16位和hashcode做异或预算，这样随机性就会很好。即便是取最后几位，也能保证有很好的均匀分布情况。同时tableSizeFor方法保证了随机长度的capacity会被整成2的整数次幂。此方法在传入initCapacity的构造方法和putEntrys批量新增时会被调用。

### 2.扩容

由于HashMap本身特殊的数据结构，它即便在默认值只有数组长度只有16的情况下，因为每个数组元素都保存的是一个链表，那它理论上就可以存储无限大的元素。但是这又引起了如果你链表的长度过长，那同样会造成寻址缓慢，这样性能就会非常差。那HashMap做出的一个优化就是空间换时间，它如果元素的阈值到达了一定界限的时候，会进行数组扩容，然后将元素重新hash放置。这样就保证了每次通过hash寻址到具体数组下标然后遍历下标上那个链表的长度都能控制在一定范围内。以此控制时间。优化性能。
HashMap具体的扩容方法叫resize()。我们只要看哪几个地方再调用这个方法，就能知道它在哪些情况下会进行扩容。

如上图，可以知道在批量增加元素的时、增加单个元素的putVal方法中（数组初始化和size大于threshold各一次）、在树状化的时候（java8的一个优化，在一定情况下会进行树状化操作将原本的链表转化为红黑树）等这些情况下都进行了扩容机制。threshold是HashMap自己维护的一个属性，用来作为触发扩容的依据，跟下代码，很容易看到threshold是在初始化调resize方法中进行的第一次赋值。如下：

第一次初始化时赋值了容量和扩容阈值，默认是defaultLoadFactor*defaultInitialCapacity也就是16乘0.75等于12。同时正常情况下走上面 newCap = oldCap << 1这条分支将数组容量扩为两倍，再在下面新new一个Node(newCap)的数组作为扩容后的数组。之后遍历旧数组将元素转移到新数组中。

### 3.树状化

HashMap中具体方法叫treeifyBin(Node<K,V>[] tab, int hash)

由上图可以看到，它在增加元素等一些操作的时候，会选择将链表树状化。但是在树状话的时候，又会去判断length是否是小于64，如果小于64的话，他认为扩容可能比树状化更有意义。
而跟下代码，很容易知道在链表长度到达8的节点时，会调用此方法。

### 4.线程不安全

最简单的一点，size是类变量而非局部变量，多个线程操作putVal的时候，++size这个操作肯定是线程不安全的。除此之外，假如两个线程对同一个map进行put操作，走到putVal方法时，hash计算出的数组下标又恰好一致（产生hash碰撞），此时继续往下走，走到循环binCount中，A线程挂起，而B线程put成功。之后A线程被唤醒继续执行，此时p.next = new Node(hash...)就会覆盖掉B线程新增的数据。从而造成数据被覆盖。 当然jdk7中还有造成循环链表、元素丢失的情况。主要是jdk7中有个transfer方法，他的链表头插入法会有一些问题。

### 5.总结

> !整个HashMap还是有一定的设计的，他的扰动函数和扩容阈值这些都是有说法的。而自己手撸的hash集合按部就班的来用数组实现的话就会发现性能这些方面的问题。


```java
package com.honghongwen.map;

import java.util.Objects;

public class Map<K, V> implements IMap<K, V> {

    private static final int DEFAULT_SIZE = 10;

    private Element[] elements;

    private int size;

    public Map() {
        elements = new Element[DEFAULT_SIZE];
    }

    @SuppressWarnings("unchecked")
    @Override
    public void set(K k, V v) {
        for (int i = 0; i < elements.length; i++) {
            Element<K, V> element = elements[i];
            if (element != null) {
                if (k.equals(element.getKey())) {
                    element.setValue(v);
                    return;
                }
            }
        }

        Element<Object, Object> element = new Element<>();
        element.setKey(k);
        element.setValue(v);
        elements[size++] = element;
    }

    @SuppressWarnings("unchecked")
    @Override
    public V get(K k) {
        for (int i = 0; i < elements.length; i++) {
            Element<K, V> element = elements[i];
            if (element != null) {
                if (Objects.equals(k, element.getKey())) {
                    return element.getValue();
                }
            }
        }
        return null;
    }

    @Override
    public void delete(K k) {

    }

    @Override
    public int size() {
        return size;
    }

    private static class Element<K, V> {

        private K key;

        private V value;

        public K getKey() {
            return key;
        }

        public void setKey(K key) {
            this.key = key;
        }

        public V getValue() {
            return value;
        }

        public void setValue(V value) {
            this.value = value;
        }
    }
}
```
