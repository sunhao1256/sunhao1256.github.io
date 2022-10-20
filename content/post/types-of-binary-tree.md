---
title: "Types of Binary Tree"
date: 2022-10-19T15:31:44+08:00
tags: ['data structure']
---

The following are common types of Binary Trees. 

>以下是常见的二叉树类型

**Full Binary Tree:**

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

**Complete Binary Tree:-**

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

