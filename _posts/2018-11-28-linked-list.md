---
layout: post
title:  "链表"
date:   2018-11-28
excerpt:  "本文是笔者总结链表的笔记"
tag:
- Algorithm
comments: true
---

> 最近笔者在刷算法题，Leetcode上链表分类的已经刷完，本文算是一个小小的总结。

## 数据结构

链表是一种离散型的存储数据结构，通过指针将节点连接起来。 下面给出一个自己实现的单链表，也是Leetcode上707 design linked list的参考答案。

```
class MyLinkedList {
public:
    /** Initialize your data structure here. */
    MyLinkedList() {
        head = tail = NULL;
        size = 0;
    }
    
    /** Get the value of the index-th node in the linked list. If the index is invalid, return -1. */
    int get(int index) {
        if (index < 0 || index >= size) { return -1; }
        Node *cur = head;
        for(int i = 0; i < index; ++i) { cur = cur->next; }
        return cur->val;
    }
    
    /** Add a node of value val before the first element of the linked list. After the insertion, the new node will be the first node of the linked list. */
    void addAtHead(int val) {
        Node *node = new Node(val, head);
        head = node;
        if (size == 0) { tail = node; }
        size += 1;
    }
    
    /** Append a node of value val to the last element of the linked list. */
    void addAtTail(int val) {
        Node *node = new Node(val, NULL);
        if (size == 0) { 
            head = tail = node; 
        } else {
            tail->next = node;
            tail = node;
        }
        size += 1;
    }
    
    /** Add a node of value val before the index-th node in the linked list. 
    If index equals to the length of linked list, the node will be appended to the end of linked list. 
    If index is greater than the length, the node will not be inserted. */
    void addAtIndex(int index, int val) {
        if (index < 0 || index > size) { return; }
        if (index == 0) { addAtHead(val); return; }
        if (index == size) { addAtTail(val); return; }
        Node *cur = head;
        for(int i = 0; i < index-1; ++i) { cur = cur->next; }
        Node *node = new Node(val, cur->next);
        cur->next = node;
        size += 1;
    }
    
    /** Delete the index-th node in the linked list, if the index is valid. */
    void deleteAtIndex(int index) {
        if (index < 0 || index >= size) { return; }
        if (index == 0) {
            head = head->next;
            size -= 1;
            return;
        }
        Node *cur = head;
        for(int i = 0; i < index-1; ++i) { cur = cur->next; }
        cur->next = cur->next->next;
        if (index == size-1) { tail = cur; }
        size -= 1;
    }

private:
    struct Node {
        int val;
        Node *next;
        Node(int x, Node* n): val(x), next(n) {}
    };
    Node *head, *tail;
    int size;
};
```

其实我们看的出来对链表的操作主要集中在对next节点的操作，因为不论是插入还是查询都需要next节点进行寻找。    

所以我们定位到一个节点需要O(n), 插入和删除一个节点则需要O(n)+O(1)。

双链表其实很像单链表，相比于单链表还有prev节点指向当前节点的上一个节点，LRU算法就用到了双链表，Leetcode 146 LRU cache就是一道使用双链表加字典的题。

除了双链表，还有循环链表，循环链表就是尾节点的next又指向了头节点，也就是此时链表成环了。

以上就是链表的所有分类了。

## 题目

关于链表的题，单链表的居多，双链表和循环链表相关的较少。不过不论哪一种链表，不论题是什么，一定是对节点的操作，一开始不熟悉可以在纸上画个例子，实例的想下就容易理解了，一旦理解了写出来就简单了。

下面笔者列举下常见的链表操作问题：

### 倒序删除

比如Leetcode 19 Remove Nth Node From End of List 就是一道要求我们删除链表中倒数第n个节点。

> 题目让我们只用一次遍历，那么就不能事先计算链表的长度了。

这个时候我们可以使用两个指针，间隔N，当后一个指针指向NULL，此时前一个指针指向的节点就是需要删除的节点。

其实当我们想到了方法后，写出来其实只需要注意节点的操作就可以，下面是笔者的答案：

```
class Solution {
public:
    ListNode* removeNthFromEnd(ListNode* head, int n) {
        ListNode *left = head;
        ListNode *right = head;
        for(int i = 0; i < n; ++i) { right = right->next; }
        if (!right) return left->next;
        while (right->next) {
            left = left->next;
            right = right->next;
        }
        left->next = left->next->next;
        return head;
    }

    ListNode* removeNthFromEndVersionTwo(ListNode* head, int n) {
        ListNode *left = head;
        ListNode *right = head;
        for(int i = 0; i < n; ++i) { right = right->next; }
        if (!right) return left->next;
        ListNode *node = NULL;
        while (right) {
            node = left;
            left = left->next;
            right = right->next;
        }
        node->next = left->next;
        return head;
    }
};
```

上述代码需要注意两个地方:

- 特殊情况

当需要删除链表中第一个节点时, 比如链表[1, 2], N = 2。此时经过初始化right已经指向NULL，因为N等于链表的长度，这个时候我们只需要返回left->next，也就相当于删除了链表中的首元素。

- 节点的处理

因为我们需要删除一个节点，为了删除我们要知道要删除的这个节点的上一个节点。

如果我们直接判断`while(right)`, 这种方式判断的话，最终left会指向要删除的节点，所以我们得用一个节点记录left的上一个节点；

但是我们如果换一种方式判断`while(right->next)`, 最终left会指向要删除节点的上一个，这样我们就不需要额外来记录上一个节点的位置了。


### 反转链表

比如Leetcode 206 Reverse Linked List 就是让我们反转一个单链表。

> 题目让我们用递归和迭代两种方式分别实现；

假设链表为1-2-3-4-5，迭代的方式就是遍历链表将其每次转为1-NULL, 2-1-NULL, ... , 5-4-3-2-1-NULL。也就是每次遍历前面增加一个原链表后面的节点，最后原链表中最后的节点就到了新链表的最前面。

```
ListNode* reverseList(ListNode* head) {
    ListNode *left = head;
    ListNode *right = NULL;
    while (head) {
        head = head->next;
        left->next = right;
        right = left;
        left = head;
    }
    return right;
}
```

递归的方式则更加简单，假设链表为1-2-3-4-5，首先我们将链表进行拆分为两部分，一部分为首节点1，另一部分为剩余节点2-3-4-5，对剩余节点进行递归，递归后的结果应当是反转好的结果，也就是5-4-3-2, 这个时候让2这个节点的next指向第一部分的节点，此时反转结束。

```
ListNode* reverseList(ListNode* head) {
    if (head == NULL || head->next == NULL) { return head; }
    ListNode *node = head->next;
    head->next = NULL;
    ListNode *newHead = reverseList(node);
    node->next = head;
    return newHead;
}
```

### 求链表的中间节点

比如Leetcode 876 Middle of the Linked List 就是让我们找出链表的中间节点。

> 题目规定当链表数量为偶数时，返回后一个为中间节点，假设链表为1-2-3-4-5-6，中间节点为3-4，所以此时返回后一个4作为中间节点

使用快慢指针，到快指针指向NULL(偶数）或者指向尾节点(奇数) 时，慢指针指向的就是中间节点了。

```
ListNode* middleNode(ListNode* head) {
    ListNode *slow = head, *fast = head;
    while(fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
    }
    return slow;
}
```

### 判断链表是否成环

判断一个链表是否成环，使用快慢指针，如果成环那么一定会相遇。也可以使用字典，将节点依次存入字典，如果有重复的节点那么就是成环。

## 总结

链表的数据结构其实不难，但是关于链表的题有时会有其他算法的加入，这个时候就考验基本功了，所以还得多总结多思考。





