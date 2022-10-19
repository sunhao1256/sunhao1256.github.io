---
title: "LinkedHashSet"
date: 2022-10-18T22:04:11+08:00
tags: ['java基础']
---

```java
package java.util;

// LinkedHashSet继承自HashSet
public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {

    private static final long serialVersionUID = -2851667679971038690L;

    // 传入容量和装载因子
    public LinkedHashSet(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor, true);
    }
    
    // 只传入容量, 装载因子默认为0.75
    public LinkedHashSet(int initialCapacity) {
        super(initialCapacity, .75f, true);
    }
    
    // 使用默认容量16, 默认装载因子0.75
    public LinkedHashSet() {
        super(16, .75f, true);
    }

    // 将集合c中的所有元素添加到LinkedHashSet中
    // 好奇怪, 这里计算容量的方式又变了
    // HashSet中使用的是Math.max((int) (c.size()/.75f) + 1, 16)
    // 这一点有点不得其解, 是作者偷懒？
    public LinkedHashSet(Collection<? extends E> c) {
        super(Math.max(2*c.size(), 11), .75f, true);
        addAll(c);
    }
    
    // 可分割的迭代器, 主要用于多线程并行迭代处理时使用
    @Override
    public Spliterator<E> spliterator() {
        return Spliterators.spliterator(this, Spliterator.DISTINCT | Spliterator.ORDERED);
    }
}


```

首先，LinkedHashSet所有的构造方法都是调用HashSet的同一个构造方法，如下：

```java
    // HashSet的构造方法
    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
```

然后，通过调用LinkedHashMap的构造方法初始化map，如下所示：

```java
    public LinkedHashMap(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor);
        accessOrder = false;
    }
```

可以看到，这里把accessOrder写死为false了。

所以，LinkedHashSet是不支持按访问顺序对元素排序的，只能按插入顺序排序。
