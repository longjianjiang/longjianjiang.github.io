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

所谓二叉搜索树就是中序遍历的结果是有序的，如下所示：

![binary_search_tree_1]({{site.url}}/assets/images/blog/binary_search_tree_1.png)

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