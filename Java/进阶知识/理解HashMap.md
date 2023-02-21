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

-   桶大小。
    -   桶数组越大，碰撞越小
-   hashcode算法。
    -   hashcode算法越平均，碰撞分配越均匀。

==Hash值的计算为：先利用类的Hashcode方法得到HashCode值，再通过Hash算法得到Hash值。==

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