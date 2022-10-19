---
title: "Hashset"
date: 2021-10-18T21:42:09+08:00
tags: ['java基础']
---

# 属性

```java
    // 内部使用HashMap
    private transient HashMap<E,Object> map;

    // 虚拟对象，用来作为value放到map中
    private static final Object PRESENT = new Object();

```

# 构造方法

```java
public HashSet() {
    map = new HashMap<>();
}

public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}

public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}

public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}

// 非public，主要是给LinkedHashSet使用的 
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}

- public :表明该成员变量或者方法是对所有类或者对象都是可见的,所有类或者对象都可以直接访问 
- private:表明该成员变量或者方法是私有的,只有当前类对其具有访问权限,除此之外其他类或者对象都没有访问权限.子类也没有访问权限. 
- protected:表明成员变量或者方法对类自身,与同在一个包中的其他类可见,其他包下的类不可访问,除非是他的子类 
- default:表明该成员变量或者方法只有自己和其位于同一个包的内可见,其他包内的类不能访问,即便是它的子类

```

# 添加元素

直接调用HashMap的put()方法，把元素本身作为key，把PRESENT作为value，也就是这个map中所有的value都是一样的。

```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

# 删除元素

直接调用HashMap的remove()方法，注意map的remove返回是删除元素的value，而Set的remov返回的是boolean类型。

这里要检查一下，如果是null的话说明没有该元素，如果不是null肯定等于PRESENT。

```java
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
```

# 查找

不需要get，这里只要一个检查元素是否存在的方法contains()，直接调用map的containsKey()方法。

```java
public boolean contains(Object o) {
    return map.containsKey(o);
}

```



# 总结

（1）HashSet内部使用HashMap的key存储元素，以此来保证元素不重复；

（2）HashSet是无序的，因为HashMap的key是无序的；

（3）HashSet中允许有一个null元素，因为HashMap允许key为null；

（4）HashSet是非线程安全的；

（5）HashSet是没有get()方法的；

# 提问

- 构造一个指定容量的hashMap如何传

  ```java
  public HashSet(Collection<? extends E> c) {
      map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
      addAll(c);
  }
  ```

  假如，我们预估HashMap要存储n个元素，那么，它的容量就应该指定为((n/0.75f) + 1)，如果这个值小于16，那就直接使用16得了。

- hashSet是怎么保证元素不重复的

  通过HashMap来保证的，因为HashMap可以保证key不重复，而hashSet使用HasMap时的value是一个定值

- hashSet有序吗

  无序，底层用HashMap，hash值&数组长度来定位下标，而iterator迭代器是的实现是对数组遍历。
