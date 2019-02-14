---
layout: post
title:  "二叉树"
date:   2019-01-26
excerpt:  "本文是笔者总结二叉树的笔记"
tag:
- Algorithm
comments: true
---

> 笔者最近在刷题时，遇到了一道关于还原二叉树的题，借此总结二叉树的基本知识。

二叉树是树形结构中一种比较简单的，父节点最多只有两个孩子节点，这两个孩子节点又可以被看作成一颗二叉树，所以二叉树的定义是一种递归的定义。

- 满二叉树，所有非叶子节点的度为2
- 完全二叉树，深度为k(k>0)，则第k-1层只会缺少右边若干连续的子节点，也就是k-1层的子节点都在左边分布

如下所示：

![binary_tree_1]({{site.url}}/assets/images/blog/binary_tree_1.png)

## 二叉树的性质

- 深度为k(k>=0)的二叉树至多有2^(k+1)-1个节点
- 二叉树第i层(根节点为第0层)至多有2^i个节点
- 二叉树中叶子节点数量为n0，度为2的节点数量为n2，则n0=n2+1

## 二叉树的存储

{% highlight cpp %}
template<typename T>
struct TreeNode {
    TreeNode<T> *parent; // optional
	TreeNode<T> *left;
	TreeNode<T> *right;
	T info;
};
{% endhighlight %}

通常我们使用上述结构体来描述二叉树的一个节点，有时为了方便会存储一个父节点。

当二叉树为完全二叉树时，我们也可以使用顺序存储的方式存储二叉树，之前说过的[堆](http://www.longjianjiang.com/heap)就是一种完全二叉树，所以建堆内部使用的存储结构时就是线性表。

## 二叉树的遍历

```
		 1
	2		  5
 3	   4    NULL   6
```


所谓二叉树的遍历就是将其转换为线型的表示方式，有四种遍历方式：

- 前序遍历

按 **根节点-左子树-右子树** 的方式递归的遍历

遍历前面二叉树打印结果为: **1 2 3 4 5 6**
- 中序遍历

按 **左子树-根节点-右子树** 的方式递归的遍历

遍历前面二叉树打印结果为: **3 2 4 1 5 6**

- 后序遍历

按 **左子树-右子树-根节点** 的方式递归的遍历

遍历前面二叉树打印结果为: **3 4 2 6 5 1**

- 层次遍历

从二叉树的根节点所在层依次往下遍历，同一层中依次从左往右进行遍历。

遍历前面二叉树打印结果为: **1 2 5 3 4 6**

> 前三种遍历方式也叫做深度优先遍历，层次遍历也叫做广度优先遍历

### 深度优先遍历

二叉树的深度优先遍历实现方式有两种方式，下面笔者给出两种方式的实现:

- 递归实现

因为二叉树的定义就是一种递归的定义，所以遍历使用递归，十分直观和简单:

{% highlight cpp %}
void preorderTraverse(TreeNode<int>* rootTree) {
    if (rootTree != NULL) {
        cout << rootTree->info << " ";
        preorderTraverse(rootTree->left);
        preorderTraverse(rootTree->right);
    }
}

void inorderTraverse(TreeNode<int>* rootTree) {
    if (rootTree != NULL) {
        inorderTraverse(rootTree->left);
        cout << rootTree->info << " ";
        inorderTraverse(rootTree->right);
    }
}

void postorderTraverse(TreeNode<int>* rootTree) {
    if (rootTree != NULL) {
        postorderTraverse(rootTree->left);
        postorderTraverse(rootTree->right);
        cout << rootTree->info << " ";
    }
}
{% endhighlight %}

- 迭代实现

当二叉树层次很深的情况，递归有可能导致栈溢出，所以我们需要一种迭代的方式遍历二叉树:

先序遍历使用迭代的方式也比较简单，使用栈，首先将根节点入栈，然后将右节点入栈，左节点入栈，取栈顶元素直到栈为空，参考代码如下:

{% highlight cpp %}
void preorderTraverseNoneRecursive(TreeNode<int>* rootTree) {
    if (rootTree == NULL) { return; }
    stack<TreeNode<int> *> s;
    s.push(rootTree);
    while (!s.empty()) {
        TreeNode<int> *top = s.top();
        cout << top->info << " ";
        s.pop();
        if (top->right) {
            s.push(top->right);
        }
        if (top->left) {
            s.push(top->left);
        }
    }
}
{% endhighlight %}

中序遍历依然使用栈来存储节点，步骤如下:

1.指针指向根节点

2.将指针入栈，不断取左节点，重复此步骤

3.当到叶子节点，弹出栈顶节点，将指针指向栈顶节点的右节点

4.重复步骤1

参考代码如下:

{% highlight cpp %}
void inorderTraverseNoneRecursive(TreeNode<int>* rootTree) {
    if (rootTree == NULL) { return; }

    stack<TreeNode<int> *> s;
    TreeNode<int> *pointer = rootTree;
    while (!s.empty() || pointer) {
        if (pointer) {
            s.push(pointer);
            pointer = pointer->left;
        } else {
            pointer = s.top();
            s.pop();
            cout << pointer->info << " ";
            pointer = pointer->right;
        }
    }
}
{% endhighlight %}

后序遍历相对复杂点，遍历时需要设置标记位，步骤如下:

1.指针指向根节点，入栈，取左节点，直到叶子节点

2.遍历栈，每次设置两个标记位，左子树是否遍历过(leftVisited), 右子树是否遍历过(rightVisited)，初始化为false

2.1判断栈顶节点的左右子节点是否是先前指针节点(叶子节点默认是NULL)，设置标记位为true

2.2取栈顶节点，当左右子节点存在，并且对应标记位为false时，将其入栈

2.3否则弹出栈顶节点

参考代码如下:

{% highlight cpp %}
void postorderTraverseNoneRecursive(TreeNode<int>* rootTree) {
    if (rootTree == NULL) { return; }

    stack<TreeNode<int> *> s;
    TreeNode<int> *pointer = rootTree;
    while (pointer) {
        s.push(pointer);
        pointer = pointer->left;
    }

    while (!s.empty()) {
        bool leftVisited = false, rightVisited = false;
        if (pointer && s.top()->left == pointer) {
            leftVisited = true;
        } else if (pointer && s.top()->right == pointer) {
            leftVisited = true;
            rightVisited = true;
        }

        pointer = s.top();
        if (pointer->left && !leftVisited) {
            s.push(pointer->left);
        } else if (pointer->right && !rightVisited) {
            s.push(pointer->right);
        } else {
            s.pop();
            cout << pointer->info << " ";
        }
    }
}
{% endhighlight %}

### 广度优先遍历

二叉树的层次遍历实现使用队列，将二叉树的左右子树依次入队列，遍历队列即可，参考代码如下:

{% highlight cpp %}
void levelOrderTraverse(TreeNode<int>* rootTree) {
    if (rootTree == NULL) { return; }

    queue<TreeNode<int>*> q;
    TreeNode<int> *cur;
    q.push(rootTree);
    while (!q.empty()) {
        cur = q.front();
        q.pop();
        cout << cur->info << " ";
        if (cur->left) {
            q.push(cur->left);
        }
        if (cur->right) {
            q.push(cur->right);
        }
    }
}
{% endhighlight %}