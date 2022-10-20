---
title: "Types of Binary Tree"
date: 2022-10-19T15:31:44+08:00
tags: ['data structure']
---

>translated from https://www.geeksforgeeks.org/binary-tree-set-3-types-of-binary-tree/?ref=gcse

The following are common types of Binary Trees. 

>以下是常见的二叉树类型

## Full Binary Tree

 A Binary Tree is a full binary tree if every node has 0 or 2 children. The following are the examples of a full binary tree. We can also say a full binary tree is a binary tree in which all nodes except leaf nodes have two children. 

> 如果一棵树上所有节点都有0或者2个孩子节点，则称为满二叉树，下面是一个满二叉树的事例，我们也可以看到一棵满二叉树实际上是一个除了叶子节点其他节点都有2个孩子的二叉树

A full Binary tree is a special type of binary tree in which every parent node/internal node has either two or no children. It is also known as a proper binary tree.

>满二叉树是每个父节点和内部节点都有2和或者没有孩子的一种特殊二叉树类型，也被称为“适当二叉树”

```
               18
           /       \  
         15         30  
        /  \        /  \
      40    50    100   40

             18
           /    \   
         15     20    
        /  \       
      40    50   
    /   \
   30   50

               18
            /     \  
          40       30  
                   /  \
                 100   40
```

## Complete Binary Tree

 A Binary Tree is a Complete Binary Tree if all the levels are completely filled except possibly the last level and the last level has all keys as left as possible.

> 如果二叉树的除了最后一层其他都被填满，并且最后一层的节点都在左侧，则被称为完全二叉树

A complete binary tree is just like a full binary tree, but with two major differences:

 >一个完全二叉树和满二叉树很类似，但是有2个主要的区别

- Every level must be completely filled

  > 每一层必须被填满

- All the leaf elements must lean towards the left.

  > 所有的叶子必须倾斜在左侧

- The last leaf element might not have a right sibling i.e. a complete binary tree doesn’t have to be a full binary tree.

  The following are examples of Complete Binary Trees.

  > 最后的叶子节点也许没有一个右侧的兄弟姐妹借点，一个完全二叉树不一定得是满二叉树，下面是完全二叉树的示例

```
               18
           /       \  
         15         30  
        /  \        /  \
      40    50    100   40


               18
           /       \  
         15         30  
        /  \        /  \
      40    50    100   40
     /  \   /
    8   7  9 
```

Practical example of Complete Binary Tree is *Binary Heap*

>完全二叉树的实际例子是*二叉树堆*

**Binary Heap**

A Binary Heap is a Binary Tree with following properties.

1. It’s a complete tree (All levels are completely filled except possibly the last level and the last level has all keys as left as possible). This property of Binary Heap makes them suitable to be stored in an array.

2. A Binary Heap is either Min Heap or Max Heap. In a Min Binary Heap, the key at root must be minimum among all keys present in Binary Heap. The same property must be recursively true for all nodes in Binary Tree. Max Binary Heap is similar to MinHeap.

> 二叉树堆是一个二叉树有着如下的属性
>
> 1. 是一棵完全二叉树，完全二叉树的除了最后一层所有层都被填满并且最后一层的节点靠左的特性让二叉树堆能够将节点合适的存在放在数组中。
> 2. 一个二叉堆可是是最小堆或者最大堆，在最小堆里，root的key一定是所有在二叉堆里最小的，这个特性必递归有效对整个树里所有的节点。最大堆同理

**Examples of Min Heap:**

```
            10                      10
         /      \               /       \  
       20        100          15         30  
      /                      /  \        /  \
    30                     40    50    100   40
```

**How is Binary Heap represented?**
A Binary Heap is a Complete Binary Tree. A binary heap is typically represented as an array.

> 二叉堆是完全二叉树，二叉堆通常用数组来实现

- The root element will be at Arr[0].

  >根节点就放在Arr[0]

- Below table shows indexes of other nodes for the ith node, i.e., Arr[i]:

  >下表展示了第i个节点的所有索引，换句话就是Arr[i]

  | Arr[(i-1)/2] | Returns the parent node      |
  | ------------ | ---------------------------- |
  | Arr[(2*i)+1] | Returns the left child node  |
  | Arr[(2*i)+2] | Returns the right child node |

The traversal method use to achieve Array representation is **Level Order**

> 用于实现数组展示的遍历方法就是按等级排序

<img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202210191719814.png" alt="image-20221019171933588" style="zoom:50%;" />

- **Applications of Heaps:**
  **1)** [Heap Sort](https://www.geeksforgeeks.org/heap-sort/): Heap Sort uses Binary Heap to sort an array in O(nLogn) time

- > 堆排序可以维持

- **2)** Priority Queue: Priority queues can be efficiently implemented using Binary Heap because it supports insert(), delete() and extractmax(), decreaseKey() operations in O(logn) time. Binomoial Heap and Fibonacci Heap are variations of Binary Heap. These variations perform union also efficiently.

- > 使用二叉堆实现的优先级队列性能很好，因为它支持插入，删除，提取最大值，降低级别操作，二项式堆和斐波那契堆都是二叉堆的变种，这些变种执行单元也一样高效

- **3)** Graph Algorithms: The priority queues are especially used in Graph Algorithms like [Dijkstra’s Shortest Path](https://www.geeksforgeeks.org/greedy-algorithms-set-7-dijkstras-algorithm-for-adjacency-list-representation/) and[ Prim’s Minimum Spanning Tree](https://www.geeksforgeeks.org/greedy-algorithms-set-5-prims-minimum-spanning-tree-mst-2/).

- **4)** Many problems can be efficiently solved using Heaps. See following for example.
  a) [K’th Largest Element in an array](https://www.geeksforgeeks.org/k-largestor-smallest-elements-in-an-array/).
  b) [Sort an almost sorted array/](https://www.geeksforgeeks.org/nearly-sorted-algorithm/)
  c) [Merge K Sorted Arrays](https://www.geeksforgeeks.org/merge-k-sorted-arrays/).



- **Operations on Min Heap:**
  **1)** getMini(): It returns the root element of Min Heap. Time Complexity of this operation is O(1)

- > 获取最小值只需要返回根元素，时间复杂度是O(1)

- **2)** extractMin(): Removes the minimum element from MinHeap. Time Complexity of this Operation is O(Logn) as this operation needs to maintain the heap property (by calling heapify()) after removing root.

- > 取出最小值，从堆里删除最小的元素，时间复杂度是O(logn)，需要在删除根后，操作堆进行堆化

- **3)** decreaseKey(): Decreases value of key. The time complexity of this operation is O(Logn). If the decreases key value of a node is greater than the parent of the node, then we don’t need to do anything. Otherwise, we need to traverse up to fix the violated heap property.

- > 降低key的级别，如果key比父亲大，则不动，否则需要向上迭代，修复不满足的堆属性

- **4)** insert(): Inserting a new key takes O(Logn) time. We add a new key at the end of the tree. IF new key is greater than its parent, then we don’t need to do anything. Otherwise, we need to traverse up to fix the violated heap property.

- > 插入一个新元素需要O(logn)的时间，我们把新元素放到堆尾部，如果比父亲大，则我们不需要操作，否则上下迭代修复不满足的元素

- **5)** delete(): Deleting a key also takes O(Logn) time. We replace the key to be deleted with minum infinite by calling decreaseKey(). After decreaseKey(), the minus infinite value must reach root, so we call extractMin() to remove the key.

- > 删除一个元素也需要O(logn)的时间，我使用负无穷值调用decreaseKey方法，最后这个元素一定是在根，然后调用extractMin拿掉这个元素就行了

## Balanced Binary Tree

A binary tree is balanced if the height of the tree is O(Log n) where n is the number of nodes. For Example, the AVL tree maintains O(Log n) height by making sure that the difference between the heights of the left and right subtrees is at most 1. Red-Black trees maintain O(Log n) height by making sure that the number of Black nodes on every root to leaf paths is the same and there are no adjacent red nodes. Balanced Binary Search trees are performance-wise good as they provide O(log n) time for search, insert and delete. 

>一个元素是n个的，并且树高度是O(logn)的树就是平衡二叉树。例如，平衡二叉树通过去报左右子树高度不超过1来保证O(logn)高度。红黑树通过保证所有到叶子节点路径上的黑节点数字一样，并且没有相邻的红色节点来保证O(logN)的高度，实现平衡。平衡二叉搜索树性能很高，因为查找，插入和删除都能达到O(logn)的程度

<img src="https://pic-frank.oss-cn-beijing.aliyuncs.com/img/202210201549516.png" alt="image-20221020154918407" style="zoom:50%;" />

It is a type of binary tree in which the difference between the height of the left and the right subtree for each node is either 0 or 1. In the figure above, the root node having a value 0 is unbalanced with a depth of 2 units.

>这是一种左右子树高度不超过1的，0节点的高度超过1了

**A degenerate (or pathological) tree:-**

A Tree where every internal node has one child. Such trees are performance-wise same as linked list. 

A degenerate or pathological tree is the tree having a single child either left or right.

>一个数，如果他的所有内部节点都只有一个孩子，这种树的性能和链表一样，一个退化的/病态的树，左右一会有一个孩子

```
      10
      /
    20
     \
     30
      \
      40   
```

**Skewed Binary Tree:-**

A skewed binary tree is a pathological/degenerate tree in which the tree is either dominated by the left nodes or the right nodes. Thus, there are two types of skewed binary tree: left-skewed binary tree and right-skewed binary tree.

>不直的树是，完全由左边节点或者右边节点组成的一个退化的/病态的树，因此不直的二叉树有两种，左不直二叉树和右不直二叉树

```
      10                                           10
      /                                             \
    20                                               20
    /                                                 \
  30                                                   30
  /                                                     \
 40                                                      40
Left-Skewed Binary Tree                               Right-Skewed Binary Tree
```

