# 基本

Java中HashMap的设计是一个桶数组+链表的组合形式（Java8后为桶数组+链表或红黑树）。

## 哈希表

HashMap在初始化是，会生成一个`Node[] table`，即哈希桶数组。

```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;    //用来定位数组索引位置
        final K key;
        V value;
        Node<K,V> next;   //链表的下一个node

        Node(int hash, K key, V value, Node<K,V> next) { ... }
        public final K getKey(){ ... }
        public final V getValue() { ... }
        public final String toString() { ... }
        public final int hashCode() { ... }
        public final V setValue(V newValue) { ... }
        public final boolean equals(Object o) { ... }
}
```

在插入数据时，Java会通过计算得到的Hash值，决定将数据插入哪个Node中，如果Node已经存在，则会插入该Node的链表尾部。故HashMap的检索速度取决于：

-   桶大小：桶数组越大，碰撞越小
-   hashcode算法：hashcode算法越平均，碰撞分配越均匀。

>Hash值的计算为：先利用类的Hashcode方法得到HashCode值，再通过Hash算法得到Hash值。

```java
static final int hash(Object key) {   //jdk1.8 & jdk1.7
     int h;
     // h = key.hashCode() 为第一步 取hashCode值
     // h ^ (h >>> 16)  为第二步 高位参与运算
     return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

## 桶大小

```java
int threshold;             // 所能容纳的键值对数量极限 
final float loadFactor;    // 负载因子
int modCount;              // map结构发生变化的次数
int size;                  // 实际存在的键值对数量
```

首先，`Node[] table`的初始化长度length(_默认值是16_)，`Load factor`为负载因子(_默认值是0.75_)。`threshold`即为当前hashmap所能容纳的最大键值对数量。键值对数量超过这个数值，桶就需要扩容，扩容后的HashMap容量是之前容量的两倍。

> HashMap的桶长度，始终是2的N次方。每次扩容都是`*2`。

一般来说，如果内存空间很多而又对时间效率要求很高，可以降低负载因子Load factor的值；相反，如果内存空间紧张而对时间效率要求不高，可以增加负载因子loadFactor的值。但一般不建议修改默认值。

## 插入数据

几个要点：
1.  当map不为空时，扩容发生在插入结束后。
2.  Java8及之后，不同的Node的数据结构可能不一样。
    1.  当冲突较少时为链表，否则可能转换为红黑树。
3.  Java8及之后，数据插入链表为尾插法。即插入链表的尾部。
    1.  因为树化需要遍历链表，故无需节省这些时间。

-   Java7及之前
    1.  插入链表采用的是头插法。即插入链表头部。这是为了避免频繁遍历链表。

## 扩容

扩容，即键值对数量超过阈值时，更换底层的table。

```java
void resize(int newCapacity) {   //传入新的容量
    Entry[] oldTable = table;    //引用扩容前的Entry数组
    int oldCapacity = oldTable.length;         
    if (oldCapacity == MAXIMUM_CAPACITY) {  //扩容前的数组大小如果已经达到最大(2^30)了
        threshold = Integer.MAX_VALUE; //修改阈值为int的最大值(2^31-1)，这样以后就不会扩容了
        return;
    }
 
    Entry[] newTable = new Entry[newCapacity];  //初始化一个新的Entry数组
    transfer(newTable);                         //！！将数据转移到新的Entry数组里
    table = newTable;                           //HashMap的table属性引用新的Entry数组
    threshold = (int)(newCapacity * loadFactor);//修改阈值
}
```

-   在Java7及之前。
    
    由于是链表的数据结构，扩容时只是遍历链表，计算元素Hash值，并如同put一般将元素插入新的桶中。这里有几个注意点：
    
    1.  插入链表采用的是头插法。则会导致扩容后的链表顺序倒置。
    2.  元素迁移的过程中在多线程情境下有可能会触发死循环（无限进行链表反转）。

---

Java8之后，由于涉及红黑树，在扩容时也进行了优化，最直接的优化点即是利用扩容容量始终为2倍这个约定，移动元素无需再计算Hash值：一个Entity在扩容时，新的位置要么在**原位置**，要么在**原长度+原位置**的位置。

-   Java8的扩容过程
    1.  遍历原哈希桶，判断节点类型。
    2.  若为单节点：将节点移动到`e.hash & (newCap - 1)`的位置。
    3.  若为链表：
        1.  将Entity的hash值与`oldCap`进行与运算。若为0则为低位，否则为高位。
        2.  将低位、高位组成两个链表。其中低位链表保持原位置，高位链表插入**原长度+原位置**的位置。
    4.  若为红黑树：
        1.  将Entity的hash值与`oldCap`进行与运算。与链表一样拆解为高低位两个红黑树。并插入到分别两个哈希桶中。
        2.  此时可能会导致红黑树[退化回链表](https://www.notion.so/HashMap-945c995136e343f29a4fe418a1a68234)。

---

Java8及之后的扩容主要有以下要点：

-   JDK8在迁移元素时是正序的，不会出现链表转置的发生。
-   无需再计算每个元素的hash值。
-   扩容可能会使得元素移动到其他桶之上。
-   扩容可能会导致红黑树退化回链表。

<aside> 💡 Hashmap是没有缩容机制的。如果确实需要，则通过一个新的hashmap来接受不会被删除的k-v对。

</aside>

# 链表和红黑树

Java8中最主要的更新就是引入了红黑树。

## 过去

在Java7及之前，对于Hash值一样的Key，其会存储在同一个Hash桶的链表上，通过`equals`方法区分。

### 链表设计的问题

链表的设计，在某个Hash桶冲突过多，会导致链表变长，从而导致速度降低到`O(n)`。

## Java8

### 引入红黑树

当某支链表过长时（大于`TREEIFY_THRESHOLD`。8），会尝试将该链表转换为红黑树。

> 如果hashMap中的哈希桶大小过小（小于`MIN_TREEIFY_CAPACITY`。64），则会优先进行扩容，而非转换为红黑树。

### 退化回链表

退化回链表可能出现在两个地方：
- remove节点时。
在红黑树的root节点为空 或者root的右节点、root的左节点、root左节点的左节点为空时。
```java
if (root == null || (movable && (root.right == null || (rl = root.left) == null|| rl.left == null))) { 
	tab[index] = first.untreeify(map); // too small 
	return; 
}
```
- 扩容时。
扩容时，因为会把红黑树拆分成高低位两棵树，如果某棵树的数据量过小（小于`UNTREEIFY_THRESHOLD`，6），则会将其退化回链表。

> remove节点时，退化回链表的判断依据不依赖于`UNTREEIFY_THRESHOLD`。


# 总结

> 内容截止到最新的Java17版本。

HashMap主要在Java8时发生了很多优化：

|          | Java7-                                                      | Java8                                                                                                                      |
| -------- | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| 数据结构 | 哈希桶数组+单向链表                                         | 哈希桶数组+单向链表或红黑树                                                                                                |
| 插入链表 | 头部插入链表                                                | 尾部插入链表                                                                                                               |
| 扩容     | 链表会被倒置。<br> 一般会重新计算元素的hash值，并单独插入。 | 链表不会被倒置。<br> 不会重新计算Hash值。原有链表/红黑树拆分成两个链表/红黑树，最终一起插入。<br> 红黑树可能会退化回链表。 |
| 多线程   | 因为头插法可能导致死循环                                    | 不会导致死循环但依旧是线程不安全的。                                                                                       |
