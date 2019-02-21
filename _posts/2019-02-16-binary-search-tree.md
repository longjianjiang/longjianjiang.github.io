---
layout: post
title:  "二叉搜索树"
date:   2019-02-16
excerpt:  "本文是笔者总结二叉搜索树的笔记"
tag:
- Algorithm
comments: true
---

> 之前笔者在[二叉树](http://www.longjianjiang.com/binary-tree/)中总结了二叉树常见的一些知识，树型结构应用广泛，所以有必要继续的学习，本文笔者学习二叉搜索树的相关知识。

所谓二叉搜索树就是中序遍历的结果是有序的，任意节点 `N` 满足下面要求:

1.该节点左子树的任意一个节点的值小于 `N`

2.该节点右子树的任意一个节点的值大于 `N`

3.该节点的左右子树也满足上述要求

一颗二叉搜索如下所示：

![binary_search_tree_1]({{site.url}}/assets/images/blog/binary_search_tree_1.png)

为了保证二叉搜索树的上述特征，所以我们在插入删除时需要进行判断，保证左子树节点的值小于父节点，右子树节点的值大于父节点。

下面笔者给出一个参考二叉搜索树的实现:

{% highlight cpp %}
template<typename T>
struct TreeNode {
    TreeNode<T> *left;
    TreeNode<T> *right;
    T info;
    TreeNode(T value): info(value), left(NULL), right(NULL) {}
    TreeNode(T value, TreeNode<T> *left, TreeNode<T> *right): info(value), left(left), right(right) {}
};

class BSTree {
    TreeNode<int> *rootTree;
public:
    BSTree() : rootTree(NULL) {}

    void show() {
        _show(rootTree);
    }
    void insert(int target) {
        if (rootTree == NULL) {
            rootTree = new TreeNode<int>(target);
            return;
        }

        _insert(rootTree, target);
    }
    int find(int target) {
        if (rootTree == NULL) { return -1; }
        return -1;
    }
    void erase(int target) {
        _erase(rootTree, target);
    }

private:
    void _show(TreeNode<int>* node) {
        if (node != NULL) {
            _show(node->left);
            cout << node->info << " ";
            _show(node->right);
        }
    }
    TreeNode<int>* _erase(TreeNode<int>* node, int target) {
        if (rootTree == NULL) { return NULL; }

        if (node->info > target) {
            // remove at left
            node->left = _erase(node->left, target);

        } else if (node->info < target) {
            // remove at right
            node->right = _erase(node->right, target);

        } else {
            // remove node
            if (node->left == NULL &&
                node->right == NULL) {
                // no left & right node
                delete node;
                node = NULL;
                return NULL;

            } else if (node->left == NULL) {
                // no left node
                TreeNode<int> *right = node->right;
                delete node;
                node = NULL;
                return right;

            } else if (node->right == NULL) {
                // no right node
                TreeNode<int> *left = node->left;
                delete node;
                node = NULL;
                return left;

            } else {
                // have left & right node
                TreeNode<int> *rm = node->right;
                while (rm->left) {
                    rm = rm->left;
                }
                node->info = rm->info;
                node->right = _erase(node->right, rm->info);
            }
        }
        return node;
    }

    void _insert(TreeNode<int>* node, int target) {
        if (node->info == target) {
            // repeat
            return;
        } else if (node->info > target) {
            // insert at left
            if (node->left == NULL) {
                node->left = new TreeNode<int>(target);
                return;
            } else {
                _insert(node->left, target);
            }
        } else {
            // insert at right
            if (node->right == NULL) {
                node->right = new TreeNode<int>(target);
                return;
            } else {
                _insert(node->right, target);
            }
        }
    }
};
{% endhighlight %}

# 平衡二叉搜索树

上面的二叉搜索树根据输入数据的不同，其树形结构也是不一样的，而且二叉树搜素树的搜索效率取决于树的高度，所以为了提高搜索效率我们需要一种高效的二叉搜索树，叫做平衡二叉搜索树，理想情况下高度为 `log2N`，不过实际情况中不容易出现。

所谓平衡二叉搜索树又分好多种类，下面笔者逐个介绍。

## AVL Tree

AVL树是一种经过插入和删除节点后可以自适应的平衡二叉搜索树，可以保证树中每个节点左右子树高度差不超过1。

有了这种自适应，当输入数据为有序的时候，此二叉搜索树并不会退化为链表。

怎么做到当自适应呢，需要通过旋转节点达到平衡的效果，存在下列四种情况:

> 为了防止递归回溯，笔者为Node增加 `parent` 节点。

- LL

{% highlight cpp %}
template <class T>
void AVLTree<T>::LLR(Node<T> *parent, Node<T> *node, Node<T> *child) {
    Node<T> *grandParent = parent->parent;
    node->parent = grandParent;
    Node<T> *nodeRight = node->right;
    if (nodeRight != nullptr) { nodeRight->parent = parent; }
    node->right = parent;
    parent->parent = node;
    parent->left = nodeRight;
    if (grandParent == nullptr) {
        root = node;
    } else if (grandParent->left == parent) {
        grandParent->left = node;
    } else {
        grandParent->right = node;
    }
}
{% endhighlight %}

- RR

{% highlight cpp %}
template <class T>
void AVLTree<T>::RRR(Node<T> *parent, Node<T> *node, Node<T> *child) {
    Node<T> *grandParent = parent->parent;
    node->parent = grandParent;
    Node<T> *nodeLeft = node->left;
    if (nodeLeft != nullptr) { nodeLeft->parent = parent; }
    node->left = parent;
    parent->parent = node;
    parent->right = nodeLeft;
    if (grandParent == nullptr) {
        root = node;
    } else if (grandParent->left == parent) {
        grandParent->left = node;
    } else {
        grandParent->right = node;
    }
}
{% endhighlight %}

- LR

{% highlight cpp %}
template <class T>
void AVLTree<T>::LRR(Node<T> *parent, Node<T> *node, Node<T> *child) {
    RRR(node, child, child->right);
    LLR(parent, child, node);
}
{% endhighlight %}

- RL

{% highlight cpp %}
template <class T>
void AVLTree<T>::RLR(Node<T> *parent, Node<T> *node, Node<T> *child) {
    LLR(node, child, child->left);
    RRR(parent, child, node);
}
{% endhighlight %}


## Red-Black Tree

红黑树同样是一种平衡二叉搜索树，和AVL树不同的是，红黑树并不是通过约束左右两边子树高度来实现平衡的，而是使用来了下面的性质来约束树的平衡的。

- 节点不是黑色就是红色
- 根节点是黑色
- 红色节点的两个孩子必须是黑色
- 从任一节点到其叶子节点的路径都包含相同数量的黑色节点
- 每一个叶子节点都是黑色的(这里叶子节点是NIL)

> 最后一个性质不影响实现，知道有这条就好。

下面给出一个红黑树:

![binary_search_tree_2]({{site.url}}/assets/images/blog/binary_search_tree_2.png)

所以在红黑树进行插入和删除的时候，因为要保证上述的性质，从而需要进行颜色修改和旋转节点操作。