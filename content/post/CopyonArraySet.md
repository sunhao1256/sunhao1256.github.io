---
title: "CopyonArraySet"
date: 2021-10-18T22:16:25+08:00
tags: ['java基础']
---

# 源码

```java
public class CopyOnWriteArraySet<E> extends AbstractSet<E>
        implements java.io.Serializable {
    private static final long serialVersionUID = 5457747651344034263L;

    // 内部使用CopyOnWriteArrayList存储元素
    private final CopyOnWriteArrayList<E> al;

    // 构造方法
    public CopyOnWriteArraySet() {
        al = new CopyOnWriteArrayList<E>();
    }
    
    // 将集合c中的元素初始化到CopyOnWriteArraySet中
    public CopyOnWriteArraySet(Collection<? extends E> c) {
        if (c.getClass() == CopyOnWriteArraySet.class) {
            // 如果c是CopyOnWriteArraySet类型，说明没有重复元素，
            // 直接调用CopyOnWriteArrayList的构造方法初始化
            @SuppressWarnings("unchecked") CopyOnWriteArraySet<E> cc =
                (CopyOnWriteArraySet<E>)c;
            al = new CopyOnWriteArrayList<E>(cc.al);
        }
        else {
            // 如果c不是CopyOnWriteArraySet类型，说明有重复元素
            // 调用CopyOnWriteArrayList的addAllAbsent()方法初始化
            // 它会把重复元素排除掉
            al = new CopyOnWriteArrayList<E>();
            al.addAllAbsent(c);
        }
    }
    
    // 获取元素个数
    public int size() {
        return al.size();
    }

    // 检查集合是否为空
    public boolean isEmpty() {
        return al.isEmpty();
    }
    
    // 检查是否包含某个元素
    public boolean contains(Object o) {
        return al.contains(o);
    }

    // 集合转数组
    public Object[] toArray() {
        return al.toArray();
    }

    // 集合转数组，这里是可能有bug的，详情见ArrayList中分析
    public <T> T[] toArray(T[] a) {
        return al.toArray(a);
    }
    
    // 清空所有元素
    public void clear() {
        al.clear();
    }
    
    // 删除元素
    public boolean remove(Object o) {
        return al.remove(o);
    }
    
    // 添加元素
    // 这里是调用CopyOnWriteArrayList的addIfAbsent()方法
    // 它会检测元素不存在的时候才添加
    // 还记得这个方法吗？当时有分析过的，建议把CopyOnWriteArrayList拿出来再看看
    public boolean add(E e) {
        return al.addIfAbsent(e);
    }
    
    // 是否包含c中的所有元素
    public boolean containsAll(Collection<?> c) {
        return al.containsAll(c);
    }
    
    // 并集
    public boolean addAll(Collection<? extends E> c) {
        return al.addAllAbsent(c) > 0;
    }

    // 单方向差集
    public boolean removeAll(Collection<?> c) {
        return al.removeAll(c);
    }

    // 交集
    public boolean retainAll(Collection<?> c) {
        return al.retainAll(c);
    }
    
    // 迭代器
    public Iterator<E> iterator() {
        return al.iterator();
    }
    
    // equals()方法
    public boolean equals(Object o) {
        // 如果两者是同一个对象，返回true
        if (o == this)
            return true;
        // 如果o不是Set对象，返回false
        if (!(o instanceof Set))
            return false;
        Set<?> set = (Set<?>)(o);
        Iterator<?> it = set.iterator();

        // 集合元素数组的快照
        Object[] elements = al.getArray();
        int len = elements.length;
        
        // 我觉得这里的设计不太好
        // 首先，Set中的元素本来就是不重复的，所以不需要再用个matched[]数组记录有没有出现过
        // 其次，两个集合的元素个数如果不相等，那肯定不相等了，这个是不是应该作为第一要素先检查
        boolean[] matched = new boolean[len];
        int k = 0;
        // 从o这个集合开始遍历
        outer: while (it.hasNext()) {
            // 如果k>len了，说明o中元素多了
            if (++k > len)
                return false;
            // 取值
            Object x = it.next();
            // 遍历检查是否在当前集合中
            for (int i = 0; i < len; ++i) {
                if (!matched[i] && eq(x, elements[i])) {
                    matched[i] = true;
                    continue outer;
                }
            }
            // 如果不在当前集合中，返回false
            return false;
        }
        return k == len;
    }

    // 移除满足过滤条件的元素
    public boolean removeIf(Predicate<? super E> filter) {
        return al.removeIf(filter);
    }

    // 遍历元素
    public void forEach(Consumer<? super E> action) {
        al.forEach(action);
    }

    // 分割的迭代器
    public Spliterator<E> spliterator() {
        return Spliterators.spliterator
            (al.getArray(), Spliterator.IMMUTABLE | Spliterator.DISTINCT);
    }
    
    // 比较两个元素是否相等
    private static boolean eq(Object o1, Object o2) {
        return (o1 == null) ? o2 == null : o1.equals(o2);
    }
}

```

# 提问

- CopyOnArraySet是如何保证数据不重复

  调用了CopyOnArrayList的addIfAbsent()方法

- 有序的吗

  数组实现的，是有序的

- 安全吗

  读写分离，安全的，保证保证最终一致性，不保证强一致性
