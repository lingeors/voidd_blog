---
title: 平衡树简介
date: 2017-07-06 00:23:00
tags:
    - 二叉树
    - 数据结构
    - AVL
---
> 二叉树是一种简单的数据结构，简单通常又意味者容错性较低、可用场景较少，因此便衍生出了很多更复杂的二叉树，比如平衡树、伸展树。这些数据结构都以二叉树为基础，因此如果不了解二叉树的读者，不适合阅读

<!-- more -->

## 前言
熟悉二叉树查找树的读者可能曾经为它简单的结构和强大的功能所着迷。它确实足够简单，也足够简洁，但有个先决条件：插入的数值应该是随机的。否则二叉查找树可能会失衡（或者数值足够随机，但在不断的删除操作后，也会导致失衡），比如当我们使用一个有序数组生成一棵二叉树查找树时，实际上相当于生成了链表，查找的时间复杂度为` O(n)`，这显然和我们的预期`O(logn）`存在比较大的差距。因此，实际二叉查找树的实现一般都比较复杂，下面我们介绍一种比较常用的数据结构，它可以保证树的深度为 `(O(log n)`，这种数据结构称为 平衡树(AVL)

## AVL 简介
平衡树指 **树中任意节点的左子树和右子树高度最多相差1的二叉查找树**。
显然，平衡树实际上就是在二叉查找树上添加了限制条件，以此来避免“糟糕”情况的发生（比如，退化成链表）。
在某个插入和删除操作中，都有可能会导致树的平衡条件被打破，即某个操作可能导致节点i的左子树和右子树的高度相差2，此时我们有必要对二叉树进行必要的调整，使其恢复平衡状态，我们使用的平衡调整操作称为 **旋转**。

## AVL核心-旋转
对于一棵AVL树，当它插入一个节点导致失衡时，必定有一个点的左子树与右子树的高度相差2，我们称这个点为 **失衡点**，AVL中有四种失衡情况，我们学习AVL的重点在于掌握这四种失衡情况所对应的旋转步骤。
四种失衡情况如下所示：
![](/images/AVL/unbalance.png)
情况一：在失衡点的左儿子的左子树添加新节点（左-左情况）
情况二：在失衡点的右儿子的右子树添加新节点（右-右情况）
情况三：在失衡点的左儿子的右子树添加新节点（左-右情况）
情况四：在失衡点的右儿子的左子树添加新节点（右-左情况）

可以看到，情况一和情况二，情况三和情况四是对称关系，因此下面我们只说明情况一和情况三对应的旋转操作。对于前两种情况，采用“单旋转”操作进行调整，后面两种情况，采用“双旋转”操作进行调整。

**注：在下面所有图示中，我们使用 k1 表示平衡点，使用 k2 表示 k1 的左儿子或右儿子。使用 k3 表示 k2 的左儿子或右儿子**

### 单旋转
这里我们考虑情况一的旋转操作 “左旋”。如下所示：
![](/images/AVL/single-left-1.jpg)
在上图中，我们将 k1 的左子树设置为 k2 的右子树，k2 的右子树设置为 k1，表示为：
```c
k1.left = k2.right;
k2.right = k1;
```
同理右旋操作为：
```c
k1.right = k2.left;
k2.left = k1;
```

### 双旋转
这里我们考虑情况三的旋转操作“右左旋”。如下所示：
![](/images/AVL/right-left-rotate.png)
双旋转为两次单旋转组成，在情况三中，采用 “右旋-左旋” 的方式，表示为：
```c
// 右旋
k2.right = k3.left;
k3.left = k2;
// 左旋
k1.left = k3.right;
k3.right = k1;
```

## AVL树的实现
> 以下代码参考 [数据结构算法分析-C语言描述][1]


在AVL树中，通常的实现方法是在每个节点添加一个高度信息 height，并通过 `|Height(T.left)-Height(T.right)| >= 2` 来判断T树是否失衡。树节点可以定义如下：
```c
typedef struct AvlNode *AvlTree;
typedef struct AvlNode *Position;
struct AvlNode {
    ElementType Element;
    struct AvlNode *left;
    struct AvlNode *right;
    int height;
};
```
节点插入的核心代码如下：
```c
/**
 * 递归插入节点
 */
AvlTree insert(AvlTree T, ElementType X) {
    if (T == NULL) {
        T = createNode();
        T->height = 0;
        T->Element = X;
        T->left = T->right = NULL;
    } else if (T->Element > X) { // 插入平衡点的左边，所以只能是情况一或情况三
        T = insert(T->left, X);
        // 失衡
        if (Height(T->left) - Height(T->right) == 2) { 
            if (T->left->Element > X) {
                // 插入左儿子的左子树，执行左旋操作
                T = SingleRotateWithLeft(T);
            } else {
                // 插入左儿子的右子树，执行左右旋的双旋转操作
                T = DoubleRotateWithLeft(T);
            }
        }
    } 
    ....
}
```
左旋操作核心代码：
```c
/**
 * @param Position k1  失衡点
 * @return Postion k2 k1子树新的根结点
 */
Position SingleRotateWithLeft(Position k1) {
    Position k2;
    k2 = k1->left;
    // 旋转操作
    k1->left = k2->Right;
    k2->right = k1;

    // 更新节点高度
    k1->height = Max(Height(k1->left), Height(k1->right)) + 1;
    k2->height = Max(Height(k2->lef), k1->left) + 1;

    return k2;
}
```

## 参考
[数据结构算法分析-C语言描述][1]


[1]: https://book.douban.com/subject/1139426/