---
layout: post
category: 数据结构
date: 2018-03-03
title: 红黑树
description: 原理及算法
---

　　R-B Tree, 全称为 Red-Black Tree(红黑树), 是一种自平衡二叉查找树，可以在$$O(\log_n)$$时间内做查找，插入和删除操作
　　通过对任何一条从根到叶子的路径上各个结点着色方式的限制，红黑树确保没有一条路径会比其他路径长出两倍，因而是接近平衡的．

## 红黑树图例

![RBTree-example](/downloads/Red-black_tree_example.svg)

## 红黑树的特性

1. 每个结点或者是黑色，或者是红色．
2. 根节点是黑色.
3. 所有叶子结点都是黑色(叶子是 NIL 结点)
4. 所有红色结点的两个子结点必须为黑色(等价描述: 从每个叶子到根上的所有路径上不能有两个连续的红色节点)
5. 从任一节点到其每个叶子的所有简单路径包含相同数目的黑色节点.

　　注意到性质 4 导致了路径不能有两个相邻的红色节点．最短的可能路径全部是黑色节点，最长的可能路径有交替的红色和黑色节点．根据性质 5, 所有的到叶结点的路径有相同数目的黑色节点，这表明没有路径能多于任何其他路径的两倍长.
　　上面这些约束确保了红黑树的关键特性: 从根到叶子的最长的可能路径不多于最短的可能路径的两倍长，使得这个树大致上是平衡的．因为如插入，删除和查找等操作的最坏情况时间都要求与树的高度成比例，这个在高度上的理论上限允许红黑树在最坏情况下都是高效的．
　　从某个节点出发(不含该结点)到达一个叶结点的任意一条简单路径上的黑色节点个数称为该结点的黑高(black-hight), 记为 bh(x). 根据性质 5, 黑高的概念是明确定义的，因为从该结点出发的所有下降到其叶结点的简单路径黑结点个数都相同．于是定义红黑树的黑高为其根结点的黑高

### 引理

　　一个有 n 个内部结点的红黑树的高度至多为 $$2\lg(n + 1)$$

### 结点定义

```C++
template <typename T>
struct RBNode
{
    static const bool RED = true;
    static const bool BLACK = false;
    T value;
    bool color;
    RBNode* parent = nullptr;
    RBNode* left = nullptr;
    RBNode* right = nullptr;
};
```

## 操作

　　由于红黑树也是特化的二叉查找树，因此红黑树上的只读操作与普通二叉查找树上的只读操作相同．
　　但是插入和删除操作会改变红黑树的结构，导致其不再满足红黑树的性质．恢复红黑树的性质需要少量$$(O(\log{n}))$$的颜色变更和不超过三次树旋转(对于插入操作是两次). 插入和删除操作次数可以保持在$$O(\log{n})$$次.

### 旋转

　　insert 和 delete 操作对树做了修改，结构可能违反上面列出的红黑树的性质．为了维护这些性质，必须要改变书中某些结点的颜色以及指针结构．
　　指针结构的修改是通过旋转(ratation)来完成的，这是一种能保持二叉搜索树性质的搜索树局部操作．
　　下图给出了两种旋转操作: 左旋，右旋.

![rataion](/downloads/ratation)

　　当在某个结点 x, 上做左旋时，假设它的右孩子为 y 而不是 T.nil; x 可以为其右孩子不是 T.nil 结点的树内任意结点．左旋以 x 到 y 的链为 "支链" 进行.它使 y 成为该子树新的根结点, x 成为 y 的左孩子, y 的左孩子称为 x 的右孩子.
　　右旋操作与左旋操作对称．

　　代码实现

```C++
template<typename Key, typename Value>
void RBTree<Key, Value>::left_rotate(RBTree::node_pointer node)
{
    node_pointer y = node->right;
    node->right = y->left;
    if (y not_eq nil)
    {
        y->left->parent = node;
    }
    y->parent = node->parent;
    if (node->parent == nil)
    {
        this->root = y;
    }else if (node == node->parent->left)
    {
        node->parent->left = y;
    }else
    {
        node->parent->right = y;
    }
    y->left = node;
    node->parent = y;
}

template<typename Key, typename Value>
void RBTree<Key, Value>::right_rotate(RBTree::node_pointer node)
{
    node_pointer x = node->left;
    node->left = x->right;
    if (x not_eq nil)
    {
        x->right->parent = node;
    }
    x->parent = node->parent;
    if (x->parent == nil)
    {
        this->root = x;
    }else if (node == node->parent->right)
    {
        node->parent->right = x;
    }else
    {
        node->parent->left = x;
    }
    x->right = node;
    node->parent = x;
}
```

　　旋转操作并不是该树暴露在外面的接口，而是恢复性质时需要进行的操作．

### 插入操作

　　我们可以在$$O(\lg{n})$$时间内想一棵含 n 个结点的红黑树中插入一个新节点．
　　我们首先以二叉查找树的方法增加结点并标记为红色(如果设置为黑色，就会导致从根到叶子的路径上多一个额外的黑结点，可能会导致连续两个红色结点的冲突，可以通过颜色调换和旋转调整). 接下来进行什么样的操作取决于其他临近结点的颜色．叔结点来指一个结点的父结点的兄弟结点.

　　在插入操作后，可以注意到

1. 性质 1, 性质 3 总是保持着
2. 性质 4 只在增加红色结点，重绘黑色结点为红色，或做旋转收到威胁．
3. 性质 5 只在增加黑色结点，重绘红色结点为黑色，或做旋转收到威胁.

　　插入后的结点会遇到以下五种情形．

1. 新结点位于树的根上，这时将该结点重绘为黑色以满足性质 2. 因为他在每个路径上对黑色结点个数加一，性质 5 匹配.

    ```C++
    /** case 1: the new node is the root*/
    if (node->parent == nullptr)
    {
        node->color = Color::BLACK;
        return;
    }
    ```
2. 新结点的父结点 P 是黑色，所以性质 4, 没有失效．在这种情形下，树仍然是满足性质的，性质 5 也未受到威胁，尽管新结点 N 有两个黑色叶子子结点，但由于新结点是红色，通过它的每个子结点的路径上的就都有同通过它所取代的黑色的叶子的路径同样数目的黑色结点.

    ```C++
    /** case 2: the new node's parent node is black*/
    if (node->parent->color == Color::BLACK)
    {
        /** nothing to do*/
        return ;
    }
    ```
3. 如果父结点 P 和叔父结点 U 二者都是红色，则将 P 和 U 重绘为黑色并将祖父结点 G 重绘为红色，同时在 G 上递归进行修复过程(将 G 作为新加入的结点进行检查)

    ```C++
    /** case 3: the new node's parent and it's uncle is both red*/
    if (node->uncle() not_eq nullptr
        and node->uncle()->color == Color::RED)
    {
        node->parent->color = Color::BLACK;
        node->uncle()->color = Color::BLACK;
        node->grandparent()->color = Color::RED;
        fix_up(node->grandparent());
        return ;
    }
    ```
4. 父结点 P 是红色而叔父结点 U 是黑色或缺少，并且新结点 N 是其 P 的右子结点而 P 是 G 的左子结点，此时做左旋调换 N 和 P 的位置．

    ```C++
    /** case 4: the new node is a right node and it's parent is a left node*/
    if (node == node->parent->right
        and node->parent == node->grandparent()->left)
    {
        left_rotate(node->parent);
        node = node->left;
    }/** the new node is a left node and it's parent is a right node*/
    else if (node == node->parent->left
             and node->parent == node->grandparent()->right)
    {
        right_rotate(node->parent);
        node = node->right;
    }
    ```
5. 父结点 P 是红色而叔父结点 U 是黑色或缺少，并且新结点 N 是其 P 的左子结点而 P 是 G 的左子结点，此时做针对祖父结点 G 的一次右旋.

    ```C++
    /** case 5: the new node's parent is red and it's uncle is black or
                missing, the new node is a left node and the it's parent
                is also a left node, do the left rotate to the grandparent
                node, or the opposite, the new node is right node and
                it's parent is also a right node do the right rotate
                to the grandparent
    */
    node->parent->color = Color::BLACK;
    node->grandparent()->color = Color::RED;
    if (node == node->parent->left
        and node->parent == node->grandparent()->left)
    {
        right_rotate(node->grandparent());
    }else
    {
        left_rotate(node->grandparent());
    }
}
    ```

#### 完整代码

```C++
template<typename Key, typename Value>
bool RBTree<Key, Value>::insert(key_refer key, value_refer value)
{
    node_pointer curr_node = this->root;
    if (curr_node == nullptr)
    {
        this->root = new RBNode(key, value);
        this->root->left = nil;
        this->root->right = nil;
        this->root->color = Color::RED;
        return true;
    }
    while (curr_node->left not_eq nil and curr_node->right not_eq nil)
    {
        if (curr_node->key == key)
        {
            return false;
        }else if (curr_node->key < key)
        {
            curr_node = curr_node->right;
        }else if (curr_node->key > key)
        {
            curr_node = curr_node->left;
        }
    }
    if (curr_node->key == key)
    {
        return false;
    }else if (curr_node->key < key)
    {
        curr_node->right = new RBNode(key, value);
        curr_node->right->parent = curr_node;
        curr_node->right->left = curr_node->right = nil;
        curr_node->right->color = Color::RED;
        fix_up(curr_node->right);
    }else if (curr_node->key > key)
    {
        curr_node->left = new RBNode(key, value);
        curr_node->left ->parent = curr_node;
        curr_node->left->left = curr_node->right = nil;
        curr_node->left->color = Color::RED;
        fix_up(curr_node->left);
    }
   return true;
}

template<typename Key, typename Value>
void RBTree<Key, Value>::fix_up(RBTree::node_pointer node)
{
    /** case 1: the new node is the root*/
    if (node->parent == nullptr)
    {
        node->color = Color::BLACK;
        return;
    }
    /** case 2: the new node's parent node is black*/
    if (node->parent->color == Color::BLACK)
    {
        /** nothing to do*/
        return ;
    }
    /** case 3: the new node's parent and it's uncle is both red*/
    if (node->uncle() not_eq nullptr 
         and node->uncle()->color == Color::RED)
    {
        node->parent->color = Color::BLACK;
        node->uncle()->color = Color::BLACK;
        node->grandparent()->color = Color::RED;
        fix_up(node->grandparent());
        return ;
    }
    /** case 4: the new node is a right node and it's parent is a left node*/
    if (node == node->parent->right 
        and node->parent == node->grandparent()->left)
    {
        left_rotate(node->parent);
        node = node->left;
    }/** the new node is a left node and it's parent is a right node*/
    else if (node == node->parent->left 
             and node->parent == node->grandparent()->right)
    {
        right_rotate(node->parent);
        node = node->right;
    }
    /** case 5: the new node's parent is red and it's uncle is black or
                missing, the new node is a left node and the it's parent
                is also a left node, do the left rotate to the grandparent
                node, or the opposite, the new node is right node and
                it's parent is also a right node do the right rotate
                to the grandparent
     */
    node->parent->color = Color::BLACK;
    node->grandparent()->color = Color::RED;
    if (node == node->parent->left 
        and node->parent == node->grandparent()->left)
    {
        right_rotate(node->grandparent());
    }else
    {
        left_rotate(node->grandparent());
    }
}
```

### 删除操作

　　在二叉查找树中，实现对一个结点的删除，要么从其左子树中找出最大的元素，要么从其右子树中找出最小的元素，并把它的值转移到要删除的结点中，这样我们就可以把问题转化为删除刚才从中复制出值的结点，它必定有少于两个非叶子的儿子．
　　如果删除一个红色结点，那么所有的性质都会得到保留；当删除一个黑色结点并且其儿子结点是红色时，将这个黑色结点删除并用其红色儿子顶替，然后将其重绘为黑色结点.
　　需要注意的是当俩个结点都为黑色时，这时先把要删除的结点顶替为其儿子结点．然后讨论每一种情况.
　　以下称呼儿子为 N, 兄弟为 S, P 为 N 的父亲. $$S_L$称呼 S 的左儿子，$$S_R$$ 称呼 S 的右儿子.

1. N 是新的根，此时已经没什么可做的．从所有路径中去除了一个黑色结点，而新根是黑色的，所以性质保持这.

    ```C++
    template<typename Key, typename Value>
    void RBTree<Key, Value>::delete_case1(RBTree::node_pointer node)
    {
        /** case 1: the node is red, just delete it and return*/
        if (node->parent not_eq nullptr)
            delete_case2(node);
    }
    ```
2. S 是红色的，此时在 N 的父结点上做左旋转，把 S 转换为 N 的祖父，我们接着对调 N 的父亲和祖父的颜色，此后，所有路径上黑色结点的数目没有改变，但现在有了一个黑色的兄弟和红色的父亲，我们可以继续按情景 4, 情景 5 或情景 6 处理．

    ```C++
    template<typename Key, typename Value>
    void RBTree<Key, Value>::delete_case2(RBTree::node_pointer node)
    {
        node_pointer sibling = node->silbing();
        if (sibling->color == Color::RED)
        {
            node->parent->color = Color ::RED;
            sibling->color = Color ::BLACK;
            if (node == node->parent->left)
            {
                left_rotate(node->parent);
            }else
            {
                right_rotate(node->parent);
            }
        }
        delete_case3(node);
    }
    ```
3. N 的父亲，S 和 S 的儿子都是黑色，将 S 重绘为红色，此时从 P 开始的每条路径上的黑色结点数都少了一个，所以在 P 为根的子树上平衡起来，但是经过 P 的路径现在比不经过 P 的路径少了一个黑色结点，所以仍然违反性质 5, 因此要从情形 1 开始在 P 上做重新平衡处理.

    ```C++
    template<typename Key, typename Value>
    void RBTree<Key, Value>::delete_case3(RBTree::node_pointer node)
    {
        node_pointer silbing = node->silbing();
        if (node->parent->color == BLACK
            and silbing->color == BLACK
            and silbing->left->color == BLACK
            and silbing->right->color == BLACK)
        {
            silbing->color = Color ::RED;
            delete_case1(node->parent);
        }else
        {
            delete_case4(node);
        }
    }
    ```
4. S 和 S 的儿子为黑色，但是 N 的父亲为红色，在这种情况下，交换 S 和 P 的颜色. 这不影响通过 S 的路径的黑色结点个数，但是他在通过 N 的路径上对黑色结点数目增加了一，填补了这些路径上删除的黑色结点．

    ```C++
    template<typename Key, typename Value>
    void RBTree<Key, Value>::delete_case4(RBTree::node_pointer node)
    {
        node_pointer silbing = node->silbing();
        if (node->parent->color == RED
            and silbing->color == BLACK
            and silbing->left->color == BLACK
            and silbing->right->color == BLACK)
        {
            silbing->color = Color ::RED;
            node->parent->color = Color ::RED;
        }else
        {
            delete_case5(node);
        }
    }
    ```
5. S 是黑色，$$S_L$$ 为红色，$$S_R$$ 为黑色，而 N 是它父亲的左儿子．这种情况下在 S 上做右旋转，这样 $$S_L$$ 成为 S 的父亲和 N 的亲兄弟．接着我们交换 S 和它的新父亲的颜色．所有路径仍有相同数目的黑色结点，但是现在 N 有了一个黑色兄弟，他的右儿子是红色，所以我们进入情形 6.

    ```C++
    template<typename Key, typename Value>
    void RBTree<Key, Value>::delete_case5(RBTree::node_pointer node)
    {
        node_pointer sibling = node->silbing();
        if (sibling->color == RED)
        {
            if (node == node->parent->left
                and sibling->left->color == RED
                and sibling->right->color == BLACK)
            {
                sibling->color = RED;
                sibling->left->color = BLACK;
                right_rotate(sibling);
            }else if (node == node->parent->right
                      and sibling->left->color == BLACK
                      and sibling->right->color == RED)
            {
                sibling->color = RED;
                sibling->right->color = BLACK;
                left_rotate(sibling);
            }
        }
        delete_case6(node);
    }
    ```
6. S 是黑色，S 的右儿子是红色，而 N 是它父亲的左儿子．这时要在 N 的父亲上做左旋转，这样 S 成为 P 的父亲和 S 的右儿子的父亲．接着交换 N 的父亲和 S 的颜色，并重绘 $$S_R$$ 为黑色．

    ```C++
    template<typename Key, typename Value>
    void RBTree<Key, Value>::delete_case6(RBTree::node_pointer node)
    {
        node_pointer sibling = node->silbing();
        sibling->color = node->parent->color;
        node->parent->color = BLACK;
        if (node == node->parent->left)
        {
            sibling->right->color = BLACK;
            left_rotate(node->parent);
        }else
        {
            sibling->left->color = BLACK;
            right_rotate(node->parent);
        }
    }
    ```

完整代码见 [github](https://github.com/jzwdsb/algorithm/blob/master/RBTree.h)

<!--
> 我所有的自负都来自我的自卑.
> 所有的英雄气概都来自我内心的软弱．
> 所有的振振有词都因为心中满是怀疑.
> 我假装无情，其实是痛恨自己的深情.
> 我以为人生的意义在于四处游荡逃亡.
> 其实只是在掩饰至今没有找到愿意驻足的地方.
-->