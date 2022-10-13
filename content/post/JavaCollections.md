---
title: "JavaCollections"
date: 2021-09-26T21:50:07+08:00
tags: ['java基础']
---

# ArrayList

<img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202209262153495.png" alt="image-20220926215325156" style="zoom:50%;" />

- ArrayList实现了List，RandomAccess，Cloneable，java.io.Serializable接口
- ArrayList实现了List，实现了基础的新增，删除，遍历等操作
- ArrayList实现了RandomAccess，实现了随机读写的能力
- ArrayList实现了Cloneable，可以被克隆
- ArrayList实现了Serializable，可以被序列化

## 属性解析

```java
//默认容量
private static final int DEFAULT_CAPACITY = 10;

//默认的空数组对象，当传入的容量是0的时候使用。
private static final Object[] EMPTY_ELEMENTDATA = {};

/**
 * 空数组，传传入容量时使用，添加第一个元素的时候会重新初始为默认容量大小
 */
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

/**
 * 存储元素的数组
 */
transient Object[] elementData; // non-private to simplify nested class access

/**
 * 集合中元素的个数
 */
private int size;
```

- DEFAULT_CAPACITY：10默认容量。
- EMPTY_ELEMENTDATA：当容量是0的时候，则会将目前的存对象的数组直接指向这个对象。
- DEFAULTCAPACITY_EMPTY_ELEMENTDATA：**为了和EMPT_ELEMENTDATA区分，EMPT_ELEMENTDATA只是单纯表示当前的List是空的，而DEFAULTCAPACITY_EMPTY_ELEMENTDATA是在调用构造函数new ArrayList()时没有传容量的话，让当前的elementData指向它，等新增第一个元素的时候，才会初始化成DEFAULT_CAPACITY大小的数组。相当于是，没有传容量，但标记他并不是一个简单的空数组，而是一个没有使用的List**
- elementData：真正存放数据的地方，使用transient是为了不序列化这个属性。
- size：ArrayList的长度，而没有使用elementData的长度

## 新增方法,添加到尾部

```java
public boolean add(E e) {
    // 检查是否需要扩容
    ensureCapacityInternal(size + 1);
    // 把元素插入到最后一位
    elementData[size++] = e;
    return true;
}

private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

private static int calculateCapacity(Object[] elementData, int minCapacity) {
    // 如果是空数组DEFAULTCAPACITY_EMPTY_ELEMENTDATA，就初始化为默认大小10
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    if (minCapacity - elementData.length > 0)
        // 扩容
        grow(minCapacity);
}

private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    // 新容量为旧容量的1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    // 如果新容量发现比需要的容量还小，则以需要的容量为准
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    // 如果新容量已经超过最大容量了，则使用最大容量
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // 以新容量拷贝出来一个新数组
    elementData = Arrays.copyOf(elementData, newCapacity);
}

```

- 检查是否需要扩容
- 设置**需要最小的容量大小**为当前容量大小+1，即size+1
- 那最小目标容量与当前存放元素的数组长度进行对比
- 如果数组是DEFAULTCAPACITY_EMPTY_ELEMENTDATA，则是第一次新增对象，设置为10，这里抛出DEFAULTCAPACITY_EMPTY_ELEMENTDATA的作用，即用于区分空数组和初始化数组。进行10的初始化动作
- 然后确认最小容量是否满足，使用最小容量与当前元素数组的长度进行比较，进行扩容
- 扩容时，第一步直接将当前元素数组容量*1.5
- 如果计算出的新容量比目标需要的最小容量还小，则以目标容量为准
- 如果超过了最大容量Integer.MAX_VALUE - 8，则以Integer.MAX_VALUE - 8为准
- 然后使用新的容量拷贝一个数组出来
- 将元素添加到进数组中，时间复杂度是O(1)

## 添加到指定位置

```java
public void add(int index, E element) {
    // 检查是否越界
    rangeCheckForAdd(index);
    // 检查是否需要扩容
    ensureCapacityInternal(size + 1);
    // 将inex及其之后的元素往后挪一位，则index位置处就空出来了
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    // 将元素插入到index的位置
    elementData[index] = element;
    // 大小增1
    size++;
}

private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

```

- 时间复杂度是O(n)，需要将目标位置后面的元素都移动一个位置

## get(int index)

```java
public E get(int index) {
    // 检查是否越界
    rangeCheck(index);
    // 返回数组index位置的元素
    return elementData(index);
}

private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

E elementData(int index) {
    return (E) elementData[index];
}
```

- 检查是否越界
- 从元素数组直接取出下标对应的元素，时间复杂度是O(1)

## remove(int index)

```java
public E remove(int index) {
    // 检查是否越界
    rangeCheck(index);

    modCount++;
    // 获取index位置的元素
    E oldValue = elementData(index);
    
    // 如果index不是最后一位，则将index之后的元素往前挪一位
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    
    // 将最后一个元素删除，帮助GC
    elementData[--size] = null; // clear to let GC do its work

    // 返回旧值
    return oldValue;
}

```

- 检查是否越界
- 获取到元素
- 如果不是最后一个元素，则移动index之后的元素往前一位
- 然后删除最后一个元素，帮助GC
- 时间复杂度是O(n)，因为存在元素移动

*删除并没有减少容量！*

## remove(Object o)

```java
public boolean remove(Object o) {
    if (o == null) {
        // 遍历整个数组，找到元素第一次出现的位置，并将其快速删除
        for (int index = 0; index < size; index++)
            // 如果要删除的元素为null，则以null进行比较，使用==
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        // 遍历整个数组，找到元素第一次出现的位置，并将其快速删除
        for (int index = 0; index < size; index++)
            // 如果要删除的元素不为null，则进行比较，使用equals()方法
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

private void fastRemove(int index) {
    // 少了一个越界的检查
    modCount++;
    // 如果index不是最后一位，则将index之后的元素往前挪一位
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    // 将最后一个元素删除，帮助GC
    elementData[--size] = null; // clear to let GC do its work
}

```

- 找到第一个等于指定元素值的元素；如果是null使用==比较，如果不是则使用equals方法

- 快速删除；

## 提问

- 为什么使用size而不是elementData的数组长度

  *Size和elementData的长度不一定相等，Size是逻辑数量，elementData是真实的数组长度，一般比Size大。  elementData定义为transient的优势，自己根据size序列化真实的元素，而不是根据数组的长度序列化元素，减少了空间占用。因为删除的时候，并没有减少容量，而序列化回来的时候，如果使用elementData就会浪费空间*

  ```java
  private void writeObject(java.io.ObjectOutputStream s)
          throws java.io.IOException{
      // 防止序列化期间有修改
      int expectedModCount = modCount;
      // 写出非transient非static属性（会写出size属性）
      s.defaultWriteObject();
  
      // 写出元素个数
      s.writeInt(size);
  
      // 依次写出元素
      for (int i=0; i<size; i++) {
          s.writeObject(elementData[i]);
      }
  
      // 如果有修改，抛出异常
      if (modCount != expectedModCount) {
          throw new ConcurrentModificationException();
      }
  }
  
  private void readObject(java.io.ObjectInputStream s)
          throws java.io.IOException, ClassNotFoundException {
      // 声明为空数组
      elementData = EMPTY_ELEMENTDATA;
  
      // 读入非transient非static属性（会读取size属性）
      s.defaultReadObject();
  
      // 读入元素个数，没什么用，只是因为写出的时候写了size属性，读的时候也要按顺序来读
      s.readInt();
  
      if (size > 0) {
          // 计算容量
          int capacity = calculateCapacity(elementData, size);
          SharedSecrets.getJavaOISAccess().checkArray(s, Object[].class, capacity);
          // 检查是否需要扩容
          ensureCapacityInternal(size);
          
          Object[] a = elementData;
          // 依次读取元素到数组中
          for (int i=0; i<size; i++) {
              a[i] = s.readObject();
          }
      }
  }
  
  ```

# CopyOnArrayList

CopyOnWriteArrayList是ArrayList的线程安全版本，内部也是通过数组实现，每次对数组的修改都完全拷贝一份新的数组来修改，修改完了再替换掉老数组，这样保证了只阻塞写操作，不阻塞读操作，实现读写分离。

<img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202209262300001.png" alt="image-20220926230004955" style="zoom:50%;" />

CopyOnWriteArrayList实现了List, RandomAccess, Cloneable, java.io.Serializable等接口。

CopyOnWriteArrayList实现了List，提供了基础的添加、删除、遍历等操作。

CopyOnWriteArrayList实现了RandomAccess，提供了随机访问的能力。

CopyOnWriteArrayList实现了Cloneable，可以被克隆。

CopyOnWriteArrayList实现了Serializable，可以被序列化。

## 属性

```java
/** 用于修改时加锁 */
final transient ReentrantLock lock = new ReentrantLock();

/** 真正存储元素的地方，只能通过getArray()/setArray()访问 */
private transient volatile Object[] array;
```

- lock用于修改时加锁，transient避免序列化
- array真正存储元素的地方，transient避免序列化，volatile表示一个线程需改后，另一个线程立刻可以看到。

## add(E e)方法

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        // 获取旧数组
        Object[] elements = getArray();
        int len = elements.length;
        // 将旧数组元素拷贝到新数组中
        // 新数组大小是旧数组大小加1
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 将元素放在最后一位
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        // 释放锁
        lock.unlock();
    }
}

```

## remove(int Index)

```java
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        // 获取旧数组
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;
        if (numMoved == 0)
            // 如果移除的是最后一位
            // 那么直接拷贝一份n-1的新数组, 最后一位就自动删除了
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            // 如果移除的不是最后一位
            // 那么新建一个n-1的新数组
            Object[] newElements = new Object[len - 1];
            // 将前index的元素拷贝到新数组中
            System.arraycopy(elements, 0, newElements, 0, index);
            // 将index后面(不包含)的元素往前挪一位
            // 这样正好把index位置覆盖掉了, 相当于删除了
            System.arraycopy(elements, index + 1, newElements, index,
                             numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```

## Size()方法

```java
public int size() {
    // 获取元素个数不需要加锁
    // 直接返回数组的长度
    return getArray().length;
}
```



## 总结

（1）CopyOnWriteArrayList使用ReentrantLock重入锁加锁，保证线程安全；

（2）CopyOnWriteArrayList的写操作都要先拷贝一份新数组，在新数组中做修改，修改完了再用新数组替换老数组，所以空间复杂度是O(n)，性能比较低下；

（3）CopyOnWriteArrayList的读操作支持随机访问，时间复杂度为O(1)；

（4）CopyOnWriteArrayList采用读写分离的思想，读操作不加锁，写操作加锁，且写操作占用较大内存空间，所以适用于读多写少的场合；

（5）CopyOnWriteArrayList只保证最终一致性，不保证实时一致性；（为什么呢？因为只有写的时候才加锁，读的时候没锁，所以不能实时一致性）

***为什么CopyOnWriteArrayList没有size属性？***

因为每次修改都是拷贝一份正好可以存储目标个数元素的数组，所以不需要size属性了，数组的长度就是集合的大小，而不像ArrayList数组的长度实际是要大于集合的大小的。

比如，add(E e)操作，先拷贝一份n+1个元素的数组，再把新元素放到新数组的最后一位，这时新数组的长度为len+1了，也就是集合的size了。

# HashMap

HashMap采用key/value存储结构，每个key对应唯一的value，查询和修改的速度都很快，能达到O(1)的平均时间复杂度。它是非线程安全的，且不保证元素存储的顺序；

<img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202209271528623.png" alt="image-20220927152853506" style="zoom:50%;" />

HashMap实现了Cloneable，可以被克隆。

HashMap实现了Serializable，可以被序列化。

HashMap继承自AbstractMap，实现了Map接口，具有Map的所有功能。

<img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202209271529689.png" alt="image-20220927152953647" style="zoom:40%;" />

在Java中，HashMap的实现采用了（数组 + 链表 + 红黑树）的复杂结构，数组的一个元素又称作桶。

在添加元素时，会根据hash值算出元素在数组中的位置，如果该位置没有元素，则直接把元素放置在此处，如果该位置有元素了，则把元素以链表的形式放置在链表的尾部。

当一个链表的元素个数达到一定的数量（且数组的长度达到一定的长度）后，则把链表转化为红黑树，从而提高效率。

数组的查询效率为O(1)，链表的查询效率是O(k)，红黑树的查询效率是O(log k)，k为桶中的元素个数，所以当元素数量非常多的时候，转化为红黑树能极大地提高效率。

## 属性

```java

/**
 * 默认的初始容量为16
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

/**
 * 最大的容量为2的30次方
 */
static final int MAXIMUM_CAPACITY = 1 << 30;

/**
 * 默认的装载因子
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;

/**
 * 当一个桶中的元素个数大于等于8时进行树化
 */
static final int TREEIFY_THRESHOLD = 8;

/**
 * 当一个桶中的元素个数小于等于6时把树转化为链表
 */
static final int UNTREEIFY_THRESHOLD = 6;

/**
 * 当桶的个数达到64的时候才进行树化
 */
static final int MIN_TREEIFY_CAPACITY = 64;

/**
 * 数组，又叫作桶（bucket）
 */
transient Node<K,V>[] table;

/**
 * 作为entrySet()的缓存
 */
transient Set<Map.Entry<K,V>> entrySet;

/**
 * 元素的数量
 */
transient int size;

/**
 * 修改次数，用于在迭代的时候执行快速失败策略
 */
transient int modCount;

/**
 * 当桶的使用数量达到多少时进行扩容，threshold = capacity * loadFactor
 */
int threshold;

/**
 * 装载因子
 */
final float loadFactor;

```

## 构造函数

```java
public HashMap(int initialCapacity, float loadFactor) {
    //检查初始化容量是否合法 
  	if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
  			//如果是超过最大，则直接用最大
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
  			//检查装载因子是否合法
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
  			//将装载因子向上取最近的2的n次方
        this.threshold = tableSizeFor(initialCapacity);
    }
}
```

## Put方法

```java
public V put(K key, V value) {
    // 调用hash(key)计算出key的hash值
    return putVal(hash(key), key, value, false, true);
}

static final int hash(Object key) {
    int h;
    // 如果key为null，则hash值为0，否则调用key的hashCode()方法
    // 并让高16位与整个hash异或，这样做是为了使计算出的hash更分散
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K, V>[] tab;
    Node<K, V> p;
    int n, i;
    // 如果桶的数量为0，则初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        // 调用resize()初始化
        n = (tab = resize()).length;
    // (n - 1) & hash 计算元素在哪个桶中
    // 如果这个桶中还没有元素，则把这个元素放在桶中的第一个位置
    if ((p = tab[i = (n - 1) & hash]) == null)
        // 新建一个节点放在桶中
        tab[i] = newNode(hash, key, value, null);
    else {
        // 如果桶中已经有元素存在了
        Node<K, V> e;
        K k;
        // 如果桶中第一个元素的key与待插入元素的key相同，保存到e中用于后续修改value值
        if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            // 如果第一个元素是树节点，则调用树节点的putTreeVal插入元素
            e = ((TreeNode<K, V>) p).putTreeVal(this, tab, hash, key, value);
        else {
            // 遍历这个桶对应的链表，binCount用于存储链表中元素的个数
            for (int binCount = 0; ; ++binCount) {
                // 如果链表遍历完了都没有找到相同key的元素，说明该key对应的元素不存在，则在链表最后插入一个新节点
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 如果插入新节点后链表长度大于8，则判断是否需要树化，因为第一个元素没有加到binCount中，所以这里-1
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果待插入的key在链表中找到了，则退出循环
                if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 如果找到了对应key的元素
        if (e != null) { // existing mapping for key
            // 记录下旧值
            V oldValue = e.value;
            // 判断是否需要替换旧值
            if (!onlyIfAbsent || oldValue == null)
                // 替换旧值为新值
                e.value = value;
            // 在节点被访问后做点什么事，在LinkedHashMap中用到
            afterNodeAccess(e);
            // 返回旧值
            return oldValue;
        }
    }
    // 到这里了说明没有找到元素
    // 修改次数加1
    ++modCount;
    // 元素数量加1，判断是否需要扩容
    if (++size > threshold)
        // 扩容
        resize();
    // 在节点插入后做点什么事，在LinkedHashMap中用到
    afterNodeInsertion(evict);
    // 没找到元素返回null
    return null;
}

```

## resize()

```java
final Node<K, V>[] resize() {
    // 旧数组
    Node<K, V>[] oldTab = table;
    // 旧容量
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    // 旧扩容门槛
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            // 如果旧容量达到了最大容量，则不再进行扩容
            threshold = Integer.MAX_VALUE;
            return oldTab;
        } else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 如果旧容量的两倍小于最大容量并且旧容量大于默认初始容量（16），则容量扩大为两部，扩容门槛也扩大为两倍
            newThr = oldThr << 1; // double threshold
    } else if (oldThr > 0) // initial capacity was placed in threshold
        // 使用非默认构造方法创建的map，第一次插入元素会走到这里
        // 如果旧容量为0且旧扩容门槛大于0，则把新容量赋值为旧门槛
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        // 调用默认构造方法创建的map，第一次插入元素会走到这里
        // 如果旧容量旧扩容门槛都是0，说明还未初始化过，则初始化容量为默认容量，扩容门槛为默认容量*默认装载因子
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int) (DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        // 如果新扩容门槛为0，则计算为容量*装载因子，但不能超过最大容量
        float ft = (float) newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float) MAXIMUM_CAPACITY ?
                (int) ft : Integer.MAX_VALUE);
    }
    // 赋值扩容门槛为新门槛
    threshold = newThr;
    // 新建一个新容量的数组
    @SuppressWarnings({"rawtypes", "unchecked"})
    Node<K, V>[] newTab = (Node<K, V>[]) new Node[newCap];
    // 把桶赋值为新数组
    table = newTab;
    // 如果旧数组不为空，则搬移元素
    if (oldTab != null) {
        // 遍历旧数组
        for (int j = 0; j < oldCap; ++j) {
            Node<K, V> e;
            // 如果桶中第一个元素不为空，赋值给e
            if ((e = oldTab[j]) != null) {
                // 清空旧桶，便于GC回收  
                oldTab[j] = null;
                // 如果这个桶中只有一个元素，则计算它在新桶中的位置并把它搬移到新桶中
                // 因为每次都扩容两倍，所以这里的第一个元素搬移到新桶的时候新桶肯定还没有元素
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 如果第一个元素是树节点，则把这颗树打散成两颗树插入到新桶中去
                    ((TreeNode<K, V>) e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 如果这个链表不止一个元素且不是一颗树
                    // 则分化成两个链表插入到新的桶中去
                    // 比如，假如原来容量为4，3、7、11、15这四个元素都在三号桶中
                    // 现在扩容到8，则3和11还是在三号桶，7和15要搬移到七号桶中去
                    // 也就是分化成了两个链表
                    Node<K, V> loHead = null, loTail = null;
                    Node<K, V> hiHead = null, hiTail = null;
                    Node<K, V> next;
                    do {
                        next = e.next;
                        // (e.hash & oldCap) == 0的元素放在低位链表中
                        // 比如，3 & 4 == 0
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        } else {
                            // (e.hash & oldCap) != 0的元素放在高位链表中
                            // 比如，7 & 4 != 0
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 遍历完成分化成两个链表了
                    // 低位链表在新桶中的位置与旧桶一样（即3和11还在三号桶中）
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 高位链表在新桶中的位置正好是原来的位置加上旧容量（即7和15搬移到七号桶了）
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}

```



## 总结

（1）HashMap是一种散列表，采用（数组 + 链表 + 红黑树）的存储结构；

（2）HashMap的默认初始容量为16（1<<4），默认装载因子为0.75f，容量总是2的n次方；

（3）HashMap扩容时每次容量变为原来的两倍；

（4）当桶的数量小于64时不会进行树化，只会扩容；

（5）当桶的数量大于64且单个桶中元素的数量大于8时，进行树化；

（6）当单个桶中元素数量小于6时，进行反树化；

（7）HashMap是非线程安全的容器；

（8）HashMap查找添加元素的时间复杂度都为O(1)；

## 提问

- 如何解决hash冲突的

  - 首先hashmap的第一层还是数组，如果只通过下标就能够将元素存下去，那么通过下标查找的话，只需要O(1)。得到下标很简单，模取长度即可。例如长度是4,用某个值直接模取长度即可，这个值就是每个对象的hashcode。例如5%4=1,6%4=2，环形数组也是这个理念。与其叫桶，不如叫环形数组。
  - 然而%这个符号运算耗费的时间相较于 位运算大很多。因此java开发者，把%改为，&(size-1)。例如5^(4-1)=1。即hash^(size-1)，当然用与运算来计算模取的值前提条件是size得是2的n次方。这也就是为什么hashmap的容量必须是2的n次方的原因。

  - hash值并不直接等于hashcode。如果直接等于hashcode的话，由于hashcode是int类型，而刚开始没有必要浪费大量内存去建大的数组，因此默认的数组空间为1<<4，16。那么假设hashcode得到的值都是前16位有变化，而后16位一直，那么得出的下标值就很容易碰撞。

    例如：1111 1111 0000 0000和1111 1110 0000 0000这两个hashcode进行&(16-1) 进行下标计算，得到的结果是一样的。这就发生了碰撞，从而退化成链表。

    因此为了解决这个问题，在计算下标的时候，hash值=hashcode^(hashcode >>> 16)。让hashcode与自身高16位进行异或，这样可以让数据分散的更均匀，减少后续计算下标时的冲突可能。异或操作会保持2个数值的特性，与操作会逐渐偏向于0，或操作会偏向于1

  - 当计算出的下标依然是同一个值的时候，则使用链表存放数据，单向链表。默认当链表的长度超过8且桶超过64，则使用红黑树来存放元素，当数的数据小于6则反树化。

- 为什么说重写equals也要重写hashcode，因为判断key是否相等，除了hashcode相同，还要调用equals方法

- 扩容的时机

  - 加完元素后，检查size是否大于当前的threshold，阈值。默认是容量的0.75倍。当然，如果强制写负载因子大于1的话，只会增加阈值，结果就是hash冲突，大量链表和红黑树，性能降低。

- 扩容时避免rehash的优化

  - Jdk7的时候，通过阈值得到new capacity大小后，需要将原来table中的bucket全部移到新的table中。因此需要重新模取计算下标。e.hash&(size-1)。为了减少rehash的次数。JDK8中做了优化。

    因此每次capacity扩容后是1<<capacity。相当于2倍。

    DEFAULT_CAPACITY是1>>>4 = 16

    5->0000 0000 0000 0101

    21->0000 0000 0001 0101

    当 capacity是16的时候，5的index是0101&(1111)=101=5，而21的index也是0101&(1111)=101=5。

    扩容后，capacity是32， 5的index是00101&(1111)=101=5，而21的index变成10101&(11111)=10101=21。

    21=5+16。即old index+old capacity。相当于以原来的16为界限，将原本会产生冲突的元素，分到了不同的bucket中。分散开了原来冲突的元素，减少了链表的长度，得到了优化。

# LinkedHashMap

LinkedHashMap继承于HashMap，除了拥有hashmap的全部功能之外，内部维护了一个双向链表。重写写了hashmap的几个方法，分别在插入，删除和使用元素的hook进行操作。

LinkedHashMap继承HashMap，拥有HashMap的所有特性，并且额外增加了按一定顺序访问的特性。

## 属性

```java
/**
* 双向链表头节点 
*/
transient LinkedHashMap.Entry<K,V> head;

/**
* 双向链表尾节点 
*/
transient LinkedHashMap.Entry<K,V> tail;

/**
* 是否按访问顺序排序 , 如果是true。则会将访问到的元素放到链表最后。LinkedHashMap,结尾的元素是最后一次使用的元素
*/
final boolean accessOrder;
```

## 内部类

```java
// 位于LinkedHashMap中,新增了2个属性before和after，实现双向链表
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}

// 位于HashMap中
static class Node<K, V> implements Map.Entry<K, V> {
    final int hash;
    final K key;
    V value;
    Node<K, V> next;
}
```

## 构造函数

```java
public LinkedHashMap(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor);
    accessOrder = false;
}

public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    accessOrder = false;
}

public LinkedHashMap() {
    super();
    accessOrder = false;
}

public LinkedHashMap(Map<? extends K, ? extends V> m) {
    super();
    accessOrder = false;
    putMapEntries(m, false);
}

public LinkedHashMap(int initialCapacity,
                     float loadFactor,
                     boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}

```

前四个构造方法accessOrder都等于false，说明双向链表是按插入顺序存储元素。

最后一个构造方法accessOrder从构造方法参数传入，如果传入true，则就实现了按访问顺序存储元素，这也是实现LRU缓存策略的关键。

## afterNodeInsertion(boolean evict)方法

```java
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
  //提供了一个机会删除最老的元素，可以实现LRU
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

（1）如果evict为true，且头节点不为空，且确定移除最老的元素，那么就调用HashMap.removeNode()把头节点移除（这里的头节点是双向链表的头节点，而不是某个桶中的第一个元素）；

（2）HashMap.removeNode()从HashMap中把这个节点移除之后，会调用afterNodeRemoval()方法；

（3）afterNodeRemoval()方法在LinkedHashMap中也有实现，用来在移除元素后修改双向链表，见下文；

（4）默认removeEldestEntry()方法返回false，也就是不删除元素。

## afterNodeAccess(Node<K,V> e)方法

在节点访问之后被调用，主要在put()已经存在的元素或get()时被调用，如果accessOrder为true，调用这个方法把访问到的节点移动到双向链表的末尾。

```java
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    // 如果accessOrder为true，并且访问的节点不是尾节点
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        // 把p节点从双向链表中移除
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        
        if (a != null)
            a.before = b;
        else
            last = b;
        
        // 把p节点放到双向链表的末尾
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        // 尾节点等于p
        tail = p;
        ++modCount;
    }
}

```

（1）如果accessOrder为true，并且访问的节点不是尾节点；

（2）从双向链表中移除访问的节点；

（3）把访问的节点加到双向链表的末尾；（**保证末尾为最新访问的元素**）

## 总结

（1）LinkedHashMap继承自HashMap，具有HashMap的所有特性；

（2）LinkedHashMap内部维护了一个双向链表存储所有的元素；

（3）如果accessOrder为false，则可以按插入元素的顺序遍历元素；

（4）如果accessOrder为true，则可以按访问元素的顺序遍历元素；

（5）LinkedHashMap的实现非常精妙，很多方法都是在HashMap中留的钩子（Hook），直接实现这些Hook就可以实现对应的功能了，并不需要再重写put()等方法；

（6）默认的LinkedHashMap并不会移除旧元素，如果需要移除旧元素，则需要重写removeEldestEntry()方法设定移除策略；

（7）LinkedHashMap可以用来实现LRU缓存淘汰策略；

## 提问

- LinkedHashMap是如何保证顺序的

  LinkedHashMap继承于HashMap，重写了HashMap的几个hook方法，还有newNode方法，内部使用LinkedHashMap自己的Entry作为元素节点，内部维护了before和after两个属性，这是标准的双向链表。

  可以查看Entry的iterator实现，实际上是在迭代这个双向链表。

  ```java
  final LinkedHashMap.Entry<K,V> nextNode() {
              LinkedHashMap.Entry<K,V> e = next;
              if (modCount != expectedModCount)
                  throw new ConcurrentModificationException();
              if (e == null)
                  throw new NoSuchElementException();
              current = e;
              next = e.after;
              return e;
          }
  ```

  而插入时调的newNode方法实际上是往链表里插数据

  ```java
      Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
          LinkedHashMap.Entry<K,V> p =
              new LinkedHashMap.Entry<K,V>(hash, key, value, e);
          linkNodeLast(p);
          return p;
      }
  
      private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
          LinkedHashMap.Entry<K,V> last = tail;
          tail = p;
          if (last == null)
              head = p;
          else {
              p.before = last;
              last.after = p;
          }
      }
  ```

  所以默认情况下，LinkedHashMap是按照插入顺序的，如果accessOrder为true则会在访问元素的时候，将访问的元素放在最后。满足最新访问的元素在结尾。

- 如何通过LinkedHashMap实现LRU算法。

  LRU是最近最少未使用的淘汰策略。

  可以从LinkedHashMap的插入方法时提供了机会去删除最老的元素，即最早插入的元素，这就是“最近”淘汰。而如果把accessOrder设置为true的话，则会将最新访问到的排到最后，那么之前的元素就是逐渐是不经常访问的。只要让插入的时候，把最靠前的删除即可。这就是“最少使用”的淘汰。

  因此LRU的实现是

  ```java
  static class LRU<K,V> extends LinkedHashMap<K,V>{
          private int capacity;
          @Override
          protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
            //大于容量的时候就删除最早的
             return size()>capacity;
          }
  
  
          public LRU(int initialCapacity, float loadFactor) {
            //启用访问排序
              this(initialCapacity,loadFactor,true);
          }
  
          public LRU(int initialCapacity, float loadFactor, boolean accessOrder) {
              super(initialCapacity, loadFactor,accessOrder);
              this.capacity=initialCapacity;
          }
      }
  ```

# WeakHashMap

顾明思议，弱引用的HashMap

- 强引用，平时的new，即时内存不够了，宁愿抛出OOM也不回收
- 软引用`JVM`在分配空间时，若果`Heap`空间不足，就会进行相应的`GC`，但是这次`GC`并不会收集软引用关联的对象，但是在JVM发现就算进行了一次回收后还是不足（`Allocation Failure`），`JVM`会尝试第二次`GC`，回收软引用关联的对象。
- 弱引用，在垃圾回收时，gc的时候如果这个对象只被弱引用关联（没有任何强引用关联他），那么这个对象就会被回收。
- 虚引用，gc的时候如果只有虚引用活着弱引用关联着对象，那么这个对象就会被回收。

**软（弱、虚）引用必须和一个引用队列（ReferenceQueue）一起使用，当gc回收这个软（弱、虚）引用引用的对象时，会把这个软（弱、虚）引用放到这个引用队列中。**

参考：https://www.jianshu.com/p/825cca41d962

<img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202209282134171.png" alt="image-20220928213449051" style="zoom:50%;" />



可见，WeakHashMap没有实现Clone和Serializable接口，所以不具有克隆和序列化的特性。

## 存储结构

WeakHashMap因为gc的时候会把没有强引用的key回收掉，所以注定了它里面的元素不会太多，因此也就不需要像HashMap那样元素多的时候转化为红黑树来处理了。

因此，WeakHashMap的存储结构只有（数组 + 链表）。

## 属性

```java
/**
 * 默认初始容量为16
 */
private static final int DEFAULT_INITIAL_CAPACITY = 16;

/**
 * 最大容量为2的30次方
 */
private static final int MAXIMUM_CAPACITY = 1 << 30;

/**
 * 默认装载因子
 */
private static final float DEFAULT_LOAD_FACTOR = 0.75f;

/**
 * 桶
 */
Entry<K,V>[] table;

/**
 * 元素个数
 */
private int size;

/**
 * 扩容门槛，等于capacity * loadFactor
 */
private int threshold;

/**
 * 装载因子
 */
private final float loadFactor;

/**
 * 引用队列，当弱键失效的时候会把Entry添加到这个队列中
 */
private final ReferenceQueue<Object> queue = new ReferenceQueue<>();

```

（1）容量

容量为数组的长度，亦即桶的个数，默认为16，最大为2的30次方，当容量达到64时才可以树化。

（2）装载因子

装载因子用来计算容量达到多少时才进行扩容，默认装载因子为0.75。

（3）引用队列

当弱键失效的时候会把Entry添加到这个队列中，当下次访问map的时候会把失效的Entry清除掉。

## Entry内部类

WeakHashMap内部的存储节点, 没有key属性。

```java
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
    // 可以发现没有key, 因为key是作为弱引用存到Referen类中
    V value;
    final int hash;
    Entry<K,V> next;

    Entry(Object key, V value,
          ReferenceQueue<Object> queue,
          int hash, Entry<K,V> next) {
        // 调用WeakReference的构造方法初始化key和引用队列
        super(key, queue);
        this.value = value;
        this.hash  = hash;
        this.next  = next;
    }
}

public class WeakReference<T> extends Reference<T> {
    public WeakReference(T referent, ReferenceQueue<? super T> q) {
        // 调用Reference的构造方法初始化key和引用队列
        super(referent, q);
    }
}

public abstract class Reference<T> {
    // 实际存储key的地方
    private T referent;         /* Treated specially by GC */
    // 引用队列
    volatile ReferenceQueue<? super T> queue;
    
    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }
}

```

## Put方法

```java
public V put(K key, V value) {
    // 如果key为空，用空对象代替
    Object k = maskNull(key);
    // 计算key的hash值
    int h = hash(k);
    // 获取桶
    Entry<K,V>[] tab = getTable();
    // 计算元素在哪个桶中，h & (length-1)
    int i = indexFor(h, tab.length);

    // 遍历桶对应的链表
    for (Entry<K,V> e = tab[i]; e != null; e = e.next) {
        if (h == e.hash && eq(k, e.get())) {
            // 如果找到了元素就使用新值替换旧值，并返回旧值
            V oldValue = e.value;
            if (value != oldValue)
                e.value = value;
            return oldValue;
        }
    }

    modCount++;
    // 如果没找到就把新值插入到链表的头部
    Entry<K,V> e = tab[i];
    tab[i] = new Entry<>(k, value, queue, h, e);
    // 如果插入元素后数量达到了扩容门槛就把桶的数量扩容为2倍大小
    if (++size >= threshold)
        resize(tab.length * 2);
    return null;
}

```

（1）计算hash；

这里与HashMap有所不同，HashMap中如果key为空直接返回0，这里是用空对象来计算的。

另外打散方式也不同，HashMap只用了一次异或，这里用了四次，HashMap给出的解释是一次够了，而且就算冲突了也会转换成红黑树，对效率没什么影响。

（2）计算在哪个桶中；

（3）遍历桶对应的链表；

（4）如果找到元素就用新值替换旧值，并返回旧值；

（5）如果没找到就在链表头部插入新元素；

HashMap就插入到链表尾部。

## resize方法

```java
void resize(int newCapacity) {
    // 获取旧桶，getTable()的时候会剔除失效的Entry
    Entry<K,V>[] oldTable = getTable();
    // 旧容量
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    // 新桶
    Entry<K,V>[] newTable = newTable(newCapacity);
    // 把元素从旧桶转移到新桶
    transfer(oldTable, newTable);
    // 把新桶赋值桶变量
    table = newTable;

    /*
     * If ignoring null elements and processing ref queue caused massive
     * shrinkage, then restore old table.  This should be rare, but avoids
     * unbounded expansion of garbage-filled tables.
     */
    // 如果元素个数大于扩容门槛的一半，则使用新桶和新容量，并计算新的扩容门槛
    if (size >= threshold / 2) {
        threshold = (int)(newCapacity * loadFactor);
    } else {
        // 否则把元素再转移回旧桶，还是使用旧桶
        // 因为在transfer的时候会清除失效的Entry，所以元素个数可能没有那么大了，就不需要扩容了
        expungeStaleEntries();
        transfer(newTable, oldTable);
        table = oldTable;
    }
}

private void transfer(Entry<K,V>[] src, Entry<K,V>[] dest) {
    // 遍历旧桶
    for (int j = 0; j < src.length; ++j) {
        Entry<K,V> e = src[j];
        src[j] = null;
        while (e != null) {
            Entry<K,V> next = e.next;
            Object key = e.get();
            // 如果key等于了null就清除，说明key被gc清理掉了，则把整个Entry清除
            if (key == null) {
                e.next = null;  // Help GC
                e.value = null; //  "   "
                size--;
            } else {
                // 否则就计算在新桶中的位置并把这个元素放在新桶对应链表的头部
                int i = indexFor(e.hash, dest.length);
                e.next = dest[i];
                dest[i] = e;
            }
            e = next;
        }
    }
}

```

（1）判断旧容量是否达到最大容量；

（2）新建新桶并把元素全部转移到新桶中；

（3）如果转移后元素个数不到扩容门槛的一半，则把元素再转移回旧桶，继续使用旧桶，说明不需要扩容；

（4）否则使用新桶，并计算新的扩容门槛；

（5）转移元素的过程中会把key为null的元素清除掉，所以size会变小；

## expungeStaleEntries方法

剔除失效的Entry。

```java
private void expungeStaleEntries() {
    // 遍历引用队列
    for (Object x; (x = queue.poll()) != null; ) {
        synchronized (queue) {
            @SuppressWarnings("unchecked")
            Entry<K,V> e = (Entry<K,V>) x;
            int i = indexFor(e.hash, table.length);
            // 找到所在的桶
            Entry<K,V> prev = table[i];
            Entry<K,V> p = prev;
            // 遍历链表
            while (p != null) {
                Entry<K,V> next = p.next;
                // 找到该元素
                if (p == e) {
                    // 删除该元素
                    if (prev == e)
                        table[i] = next;
                    else
                        prev.next = next;
                    // Must not null out e.next;
                    // stale entries may be in use by a HashIterator
                    e.value = null; // Help GC
                    size--;
                    break;
                }
                prev = p;
                p = next;
            }
        }
    }
}

```

（1）当key失效的时候gc会自动把对应的Entry添加到这个引用队列中；

（2）所有对map的操作都会直接或间接地调用到这个方法先移除失效的Entry，比如getTable()、size()、resize()；

（3）这个方法的目的就是遍历引用队列，并把其中保存的Entry从map中移除掉，具体的过程请看类注释；

（4）从这里可以看到移除Entry的同时把value也一并置为null帮助gc清理元素，防御性编程。

## 案例

```java
package com.coolcoding.code;

import java.util.Map;
import java.util.WeakHashMap;

public class WeakHashMapTest {

public static void main(String[] args) {
    Map<String, Integer> map = new WeakHashMap<>(3);

    // 放入3个new String()声明的字符串
    map.put(new String("1"), 1);
    map.put(new String("2"), 2);
    map.put(new String("3"), 3);

    // 放入不用new String()声明的字符串
    map.put("6", 6);

    // 使用key强引用"3"这个字符串
    String key = null;
    for (String s : map.keySet()) {
        // 这个"3"和new String("3")不是一个引用
        if (s.equals("3")) {
            key = s;
        }
    }

    // 输出{6=6, 1=1, 2=2, 3=3}，未gc所有key都可以打印出来
    System.out.println(map);

    // gc一下
    System.gc();

    // 放一个new String()声明的字符串
    map.put(new String("4"), 4);

    // 输出{4=4, 6=6, 3=3}，gc后放入的值和强引用的key可以打印出来
    System.out.println(map);

    // key与"3"的引用断裂
    key = null;

    // gc一下
    System.gc();

    // 输出{6=6}，gc后强引用的key可以打印出来
    System.out.println(map);
}
}


```



## 总结

（1）WeakHashMap使用（数组 + 链表）存储结构；

（2）WeakHashMap中的key是弱引用，gc的时候会被清除；

（3）每次对map的操作都会剔除失效key对应的Entry；

（4）使用String作为key时，一定要使用new String()这样的方式声明key，才会失效，其它的基本类型的包装类型是一样的；

（5）WeakHashMap常用来作为缓存使用；

# TreeMap

TreeMap使用红黑树存储元素，可以保证元素按key值的大小进行遍历。

## 继承体系

<img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202209292105412.png" alt="image-20220929210522264" style="zoom:50%;" />

TreeMap实现了Map、SortedMap、NavigableMap、Cloneable、Serializable等接口。

SortedMap规定了元素可以按key的大小来遍历，它定义了一些返回部分map的方法。

```java
public interface SortedMap<K,V> extends Map<K,V> {
    // key的比较器
    Comparator<? super K> comparator();
    // 返回fromKey（包含）到toKey（不包含）之间的元素组成的子map
    SortedMap<K,V> subMap(K fromKey, K toKey);
    // 返回小于toKey（不包含）的子map
    SortedMap<K,V> headMap(K toKey);
    // 返回大于等于fromKey（包含）的子map
    SortedMap<K,V> tailMap(K fromKey);
    // 返回最小的key
    K firstKey();
    // 返回最大的key
    K lastKey();
    // 返回key集合
    Set<K> keySet();
    // 返回value集合
    Collection<V> values();
    // 返回节点集合
    Set<Map.Entry<K, V>> entrySet();
}

```

NavigableMap是对SortedMap的增强，定义了一些返回离目标key最近的元素的方法。

```java
public interface NavigableMap<K,V> extends SortedMap<K,V> {
    // 小于给定key的最大节点
    Map.Entry<K,V> lowerEntry(K key);
    // 小于给定key的最大key
    K lowerKey(K key);
    // 小于等于给定key的最大节点
    Map.Entry<K,V> floorEntry(K key);
    // 小于等于给定key的最大key
    K floorKey(K key);
    // 大于等于给定key的最小节点
    Map.Entry<K,V> ceilingEntry(K key);
    // 大于等于给定key的最小key
    K ceilingKey(K key);
    // 大于给定key的最小节点
    Map.Entry<K,V> higherEntry(K key);
    // 大于给定key的最小key
    K higherKey(K key);
    // 最小的节点
    Map.Entry<K,V> firstEntry();
    // 最大的节点
    Map.Entry<K,V> lastEntry();
    // 弹出最小的节点
    Map.Entry<K,V> pollFirstEntry();
    // 弹出最大的节点
    Map.Entry<K,V> pollLastEntry();
    // 返回倒序的map
    NavigableMap<K,V> descendingMap();
    // 返回有序的key集合
    NavigableSet<K> navigableKeySet();
    // 返回倒序的key集合
    NavigableSet<K> descendingKeySet();
    // 返回从fromKey到toKey的子map，是否包含起止元素可以自己决定
    NavigableMap<K,V> subMap(K fromKey, boolean fromInclusive,
                             K toKey,   boolean toInclusive);
    // 返回小于toKey的子map，是否包含toKey自己决定
    NavigableMap<K,V> headMap(K toKey, boolean inclusive);
    // 返回大于fromKey的子map，是否包含fromKey自己决定
    NavigableMap<K,V> tailMap(K fromKey, boolean inclusive);
    // 等价于subMap(fromKey, true, toKey, false)
    SortedMap<K,V> subMap(K fromKey, K toKey);
    // 等价于headMap(toKey, false)
    SortedMap<K,V> headMap(K toKey);
    // 等价于tailMap(fromKey, true)
    SortedMap<K,V> tailMap(K fromKey);
}

```

## 属性

```java
/**
 * 比较器，如果没传则key要实现Comparable接口
 */
private final Comparator<? super K> comparator;

/**
 * 根节点
 */
private transient Entry<K,V> root;

/**
 * 元素个数
 */
private transient int size = 0;

/**
 * 修改次数
 */
private transient int modCount = 0;

```

（1）comparator

按key的大小排序有两种方式，一种是key实现Comparable接口，一种方式通过构造方法传入比较器。

（2）root

根节点，TreeMap没有桶的概念，所有的元素都存储在一颗树中。

## 内部类

```java
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    boolean color = BLACK;
}
//标准的二叉树结构，color意味着是红黑树
```

## 构造方法

```java
/**
 * 默认构造方法，key必须实现Comparable接口 
 */
public TreeMap() {
    comparator = null;
}

/**
 * 使用传入的comparator比较两个key的大小
 */
public TreeMap(Comparator<? super K> comparator) {
    this.comparator = comparator;
}
    
/**
 * key必须实现Comparable接口，把传入map中的所有元素保存到新的TreeMap中 
 */
public TreeMap(Map<? extends K, ? extends V> m) {
    comparator = null;
    putAll(m);
}

/**
 * 使用传入map的比较器，并把传入map中的所有元素保存到新的TreeMap中 
 */
public TreeMap(SortedMap<K, ? extends V> m) {
    comparator = m.comparator();
    try {
        buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
    } catch (java.io.IOException cannotHappen) {
    } catch (ClassNotFoundException cannotHappen) {
    }
}

```

## get(Object key)方法

标准的二叉树查找方法，通过compare或者comparator比较大小

```java
public V get(Object key) {
    // 根据key查找元素
    Entry<K,V> p = getEntry(key);
    // 找到了返回value值，没找到返回null
    return (p==null ? null : p.value);
}

final Entry<K,V> getEntry(Object key) {
    // 如果comparator不为空，使用comparator的版本获取元素
    if (comparator != null)
        return getEntryUsingComparator(key);
    // 如果key为空返回空指针异常
    if (key == null)
        throw new NullPointerException();
    // 将key强转为Comparable
    @SuppressWarnings("unchecked")
    Comparable<? super K> k = (Comparable<? super K>) key;
    // 从根元素开始遍历
    Entry<K,V> p = root;
    while (p != null) {
        int cmp = k.compareTo(p.key);
        if (cmp < 0)
            // 如果小于0从左子树查找
            p = p.left;
        else if (cmp > 0)
            // 如果大于0从右子树查找
            p = p.right;
        else
            // 如果相等说明找到了直接返回
            return p;
    }
    // 没找到返回null
    return null;
}
    
final Entry<K,V> getEntryUsingComparator(Object key) {
    @SuppressWarnings("unchecked")
    K k = (K) key;
    Comparator<? super K> cpr = comparator;
    if (cpr != null) {
        // 从根元素开始遍历
        Entry<K,V> p = root;
        while (p != null) {
            int cmp = cpr.compare(k, p.key);
            if (cmp < 0)
                // 如果小于0从左子树查找
                p = p.left;
            else if (cmp > 0)
                // 如果大于0从右子树查找
                p = p.right;
            else
                // 如果相等说明找到了直接返回
                return p;
        }
    }
    // 没找到返回null
    return null;
}

```

## 存储结构红黑树

<img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202209292112399.png" alt="image-20220929211210337" style="zoom:50%;" />

TreeMap只使用到了红黑树，所以它的时间复杂度为O(log n)，我们再来回顾一下红黑树的特性。

（1）每个节点或者是黑色，或者是红色。

（2）根节点是黑色。

（3）每个叶子节点（NIL）是黑色。（注意：这里叶子节点，是指为空(NIL或NULL)的叶子节点！）

（4）如果一个节点是红色的，则它的子节点必须是黑色的。

（5）从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑节点。

## 左旋

<img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202209292114015.png" alt="image-20220929211408906" style="zoom:50%;" />

```java
//左旋
private void rotateLeft(Entry<K,V> p) {
    if (p != null) {
      	//先取出当前节点的右节点
        Entry<K,V> r = p.right;
        //将有节点的左节点放到当前的右节点身上
      	p.right = r.left;
      	//如果右节点的左节点不为空的话
        if (r.left != null)
          //要将有节点的左节点切除和原父亲的关系，绑定到新的当前节点身上，即p
            r.left.parent = p;
        //将右节点移到当前节点的位置
      	//将右节点的父亲变为当前节点P的父亲，这样位置就移过来了
        r.parent = p.parent;
      	//如果p的父亲是null，说明原来的p是根节点，将根节点变为右节点
        if (p.parent == null)
            root = r;
        //如果p有父亲并且是左节点，就把有节点放到父亲的左边，否则就是右边
        else if (p.parent.left == p)
            p.parent.left = r;
        else
            p.parent.right = r;
      	//最后将当前的节点，整个放在有节点的左边。并且结上父亲关系
        r.left = p;
        p.parent = r;
    }
}		
```

## 右旋

<img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202209292115069.png" alt="image-20220929211505985" style="zoom:50%;" />

```java
//右旋   
private void rotateRight(Entry<K,V> p) {   
  if (p != null) {
    			//得到当前节点的左节点
            Entry<K,V> l = p.left;
    				//左节点的右节点，放在当前节点的左边
            p.left = l.right;
    				//链接父子关系，将左节点的右边的父亲，原本是左节点，改为当前节点
            if (l.right != null) l.right.parent = p;
    				//左节点，移到当前节点位置，首先处理父亲节点关系
            l.parent = p.parent;
    				//当前节点是null的话，说明当前节点是root，则移过来的左节点变为root
            if (p.parent == null)
                root = l;
    				//如果父亲不是null，在右边就将左节点放到右边，vice versa
            else if (p.parent.right == p)
                p.parent.right = l;
            else p.parent.left = l;
    				//最后在将当前节点放到左节点的右侧，关联上父亲关系
            l.right = p;
            p.parent = l;
        }
    }
```

## Put方法

```java
    public V put(K key, V value) {
      //找到root
        Entry<K,V> t = root;
        if (t == null) {
          //root不存在
            compare(key, key); // type (and possibly null) check

          //root就是这个元素了
            root = new Entry<>(key, value, null);
            size = 1;
            modCount++;
            return null;
        }
        int cmp;
      //root存在
        Entry<K,V> parent;
        // split comparator and comparable paths
      //默认用 comparator来比较，找到和key一样的值，替换掉旧的值
        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            do {
                parent = t;
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
      //使用comparable，要检查key不能为空，也是查整个树，有没有key值一样的，替换掉旧值
        else {
            if (key == null)
                throw new NullPointerException();
            @SuppressWarnings("unchecked")
                Comparable<? super K> k = (Comparable<? super K>) key;
            do {
                parent = t;
                cmp = k.compareTo(t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
      //没在当前树里找到值，但是已经找到parent了。parent就是上面的t
        Entry<K,V> e = new Entry<>(key, value, parent);
      //插入节点
        if (cmp < 0)
            parent.left = e;
        else
            parent.right = e;
      //平衡树  
      fixAfterInsertion(e);
        size++;
        modCount++;
        return null;
    }

```

## 插入后平衡

```java
/**
 * 插入再平衡
 *（1）每个节点或者是黑色，或者是红色。
 *（2）根节点是黑色。
 *（3）每个叶子节点（NIL）是黑色。（注意：这里叶子节点，是指为空(NIL或NULL)的叶子节点！）
 *（4）如果一个节点是红色的，则它的子节点必须是黑色的。
 *（5）从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑节点。
 */
private void fixAfterInsertion(Entry<K,V> x) {
    // 插入的节点为红节点，x为当前节点
    x.color = RED;

    // 只有当插入节点不是根节点且其父节点为红色时才需要平衡（违背了特性4）
    while (x != null && x != root && x.parent.color == RED) {
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            // a）如果父节点是祖父节点的左节点
            // y为叔叔节点
            Entry<K,V> y = rightOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                // 情况1）如果叔叔节点为红色
                // （1）将父节点设为黑色
                setColor(parentOf(x), BLACK);
                // （2）将叔叔节点设为黑色
                setColor(y, BLACK);
                // （3）将祖父节点设为红色
                setColor(parentOf(parentOf(x)), RED);
                // （4）将祖父节点设为新的当前节点
                x = parentOf(parentOf(x));
            } else {
                // 如果叔叔节点为黑色
                // 情况2）如果当前节点为其父节点的右节点
                if (x == rightOf(parentOf(x))) {
                    // （1）将父节点设为当前节点
                    x = parentOf(x);
                    // （2）以新当前节点左旋
                    rotateLeft(x);
                }
                // 情况3）如果当前节点为其父节点的左节点（如果是情况2）则左旋之后新当前节点正好为其父节点的左节点了）
                // （1）将父节点设为黑色
                setColor(parentOf(x), BLACK);
                // （2）将祖父节点设为红色
                setColor(parentOf(parentOf(x)), RED);
                // （3）以祖父节点为支点进行右旋
                rotateRight(parentOf(parentOf(x)));
            }
        } else {
            // b）如果父节点是祖父节点的右节点
            // y是叔叔节点
            Entry<K,V> y = leftOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                // 情况1）如果叔叔节点为红色
                // （1）将父节点设为黑色
                setColor(parentOf(x), BLACK);
                // （2）将叔叔节点设为黑色
                setColor(y, BLACK);
                // （3）将祖父节点设为红色
                setColor(parentOf(parentOf(x)), RED);
                // （4）将祖父节点设为新的当前节点
                x = parentOf(parentOf(x));
            } else {
                // 如果叔叔节点为黑色
                // 情况2）如果当前节点为其父节点的左节点
                if (x == leftOf(parentOf(x))) {
                    // （1）将父节点设为当前节点
                    x = parentOf(x);
                    // （2）以新当前节点右旋
                    rotateRight(x);
                }
                // 情况3）如果当前节点为其父节点的右节点（如果是情况2）则右旋之后新当前节点正好为其父节点的右节点了）
                // （1）将父节点设为黑色
                setColor(parentOf(x), BLACK);
                // （2）将祖父节点设为红色
                setColor(parentOf(parentOf(x)), RED);
                // （3）以祖父节点为支点进行左旋
                rotateLeft(parentOf(parentOf(x)));
            }
        }
    }
    // 平衡完成后将根节点设为黑色
    root.color = BLACK;
}

```

## 二叉树的遍历

```java
public class BinaryTree<K,V> {
    private Node<K,V> root;
    private final Comparator<Node<K,V>> comparator;

    public BinaryTree(Comparator<Node<K, V>> comparator) {
        this.comparator = comparator;
    }

    public void put(Node<K,V> node){
        if (root==null){
            root=node;
        }else {
            Node<K,V> parent=root,n;
            int compare;
            do {
                n=parent;
                compare = comparator.compare(parent, node);
                if (compare>0){
                    parent=parent.right;
                }else if (compare<0){
                    parent=parent.left;
                }else {
                    parent.setValue(node.getValue());
                    return;
                }
            }while (parent!=null);

            if (compare>0){
              n.right=node;
            } else {
                n.left=node;
            }
        }
    }

    public void inOrderTraverse(Integer type){
        inOrderTraverse(root,type);
    }
    private void inOrderTraverse(Node root,Integer type){
        if (root!=null){
            switch (type){
                case 2:
                    if (root.left!=null){
                        inOrderTraverse(root.left,type);
                    }
                    System.out.print(root.getKey());
                    if (root.right!=null){
                        inOrderTraverse(root.right,type);
                    }
                    break;
                case 1:
                    System.out.print(root.getKey());
                    if (root.left!=null){
                        inOrderTraverse(root.left,type);
                    }
                    if (root.right!=null){
                        inOrderTraverse(root.right,type);
                    }
                    break;
                case 3:
                    if (root.left!=null){
                        inOrderTraverse(root.left,type);
                    }
                    if (root.right!=null){
                        inOrderTraverse(root.right,type);
                    }
                    System.out.print(root.getKey());
                    break;
            }

        }
    }


    @Data
    static class Node<K,V> {
        private K key;
        private V value;
        private Node<K,V> parent;
        private Node<K,V> left;
        private Node<K,V> right;

        public Node(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }


    public static void main(String[] args) {
        BinaryTree<Integer, Integer> integerBinaryTree = new BinaryTree<>(Comparator.comparingInt(Node::getKey));

        integerBinaryTree.put(new Node<>(2,1));
        integerBinaryTree.put(new Node<>(6,2));
        integerBinaryTree.put(new Node<>(3,3));
        integerBinaryTree.put(new Node<>(6,4));
        integerBinaryTree.put(new Node<>(9,5));
        integerBinaryTree.put(new Node<>(7,5));
        integerBinaryTree.put(new Node<>(8,5));
        integerBinaryTree.put(new Node<>(4,5));
        integerBinaryTree.put(new Node<>(10,5));

        System.out.print("前序遍历:");
        integerBinaryTree.inOrderTraverse(1);
        System.out.println();
        System.out.print("中序遍历:");
        integerBinaryTree.inOrderTraverse(2);
        System.out.println();
        System.out.print("后序遍历:");
        integerBinaryTree.inOrderTraverse(3);
        System.out.println();
    }
}

```

## 遍历方法

遍历方式不优雅，我们使用的递归，如果元素很多的话，会导致递归调很多的方法，栈太多，最后导致内存溢出，而默认的方法有优化

```java
@Override
public void forEach(BiConsumer<? super K, ? super V> action) {
    Objects.requireNonNull(action);
    // 遍历前的修改次数
    int expectedModCount = modCount;
    // 执行遍历，先获取第一个元素的位置，再循环遍历后继节点
    for (Entry<K, V> e = getFirstEntry(); e != null; e = successor(e)) {
        // 执行动作
        action.accept(e.key, e.value);

        // 如果发现修改次数变了，则抛出异常
        if (expectedModCount != modCount) {
            throw new ConcurrentModificationException();
        }
    }
}


    final Entry<K,V> getFirstEntry() {
        Entry<K,V> p = root;
        // 从根节点开始找最左边的节点，即最小的元素
        if (p != null)
            while (p.left != null)
                p = p.left;
        return p;
    }


static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
    if (t == null)
        // 如果当前节点为空，返回空
        return null;
    else if (t.right != null) {
        // 如果当前节点有右子树，取右子树中最小的节点
        Entry<K,V> p = t.right;
        while (p.left != null)
            p = p.left;
        return p;
    } else {
        // 如果当前节点没有右子树
        // 如果当前节点是父节点的左子节点，直接返回父节点
        // 如果当前节点是父节点的右子节点，一直往上找，直到找到一个祖先节点是其父节点的左子节点为止，返回这个祖先节点的父节点
        Entry<K,V> p = t.parent;
        Entry<K,V> ch = t;
        while (p != null && ch == p.right) {
            ch = p;
            p = p.parent;
        }
        return p;
    }
}

```

## 总结

（1）TreeMap的存储结构只有一颗红黑树；

（2）TreeMap中的元素是有序的，按key的顺序排列；

（3）TreeMap比HashMap要慢一些，因为HashMap前面还做了一层桶，寻找元素要快很多；

（4）TreeMap没有扩容的概念；

（5）TreeMap的遍历不是采用传统的递归式遍历；

（6）TreeMap可以按范围查找元素，查找最近的元素；

# HashMap，LinkedHashMap，TreeMap的使用场景

- HashMap，在不考虑冲突的情况下，CRUD的时间复杂度近乎O(1)。适合只进行CRUD的场景
- 如果需要保证插入的顺序，以及遍历的顺序，则可以使用LinkedHashMap。如果通过compare查询的，最坏需要O(n)
- 如果需要查询很快，则使用二叉树，二叉树里使用红黑树能解决普通二叉树退化为链表和平衡二叉树在频繁插入删除下的频繁平衡问题，时间复杂度均维持在O(logN)，那么就可以使用TreeMap，插入和删除后，维持原来的排序，LinkedHashMap，需要O(n)。

