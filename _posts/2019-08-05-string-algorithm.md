---
layout: post
title: "字符串算法"
date: 2019-08-05
excerpt: "nope"
tag:
- Algorithm
comments: true
---

> 笔者在这里记录下字符串相关的算法。

# KMP

给定字符串s和子串p，让我们寻找p在s中的位置。

使用传统的暴力方法，我们使用两个指针来进行遍历，假设i指向s，j指向p。当s[i] != p[j] 的时候，此时i和j都要回退进行下一次的寻找。所以当s很长的时候，就很浪费时间。下面给出一个参考代码：

{% highlight cpp %}
int find_brute(string s, string p) {
    int i = 0, j = 0;
    int sl = (int)s.size(), pl = (int)p.size();

    while (i < sl && j < pl) {
        if (s[i] == p[j]) {
            i++; j++;
        } else {
            i = i - (j-1);
            j = 0;
        }
    }
    if (j == pl) {
        return i - j;
    } else {
        return -1;
    }
}
{% endhighlight %}

如何利用已经比较过的信息呢？之前的暴力方案中，当遇到某个字符不匹配的时候，我们将i进行回退，当子串存在重复字符的时候，这样简单的将i进行回退会造成不必要的比较。

所以KMP的一个目的就在于，i不会往前移，具体是如何执行的呢？有两种思路：

## next数组

```
BBC ABCDAB ABCDABCD...
    ABCDABD
```

此时i=10， j=6，s[i] != p[j]; 如果我们将i=5，j=0重新比较时，肯定会失败，因为p[5]==p[1]; i=6，i=7也是一样的，必然会不匹配。

p是`ABCDABD`，可以发现存在重复字符，在看之前其实我们已经匹配了`ABCDAB`了，所以我们可以直接将p[2] 和 s[10]进行比较，因为s[10]前面两个字符是AB，且p前两个字符也是AB。

```
BBC ABCDAB ABCDABCD...
        ABCDABD
```

根据上面我们大概知道，当越到不匹配的时候，需要去更新j，i保持不动即可。还是前面的例子i=10，j=6时字符不匹配，接下来可以将j=2进行下一次的匹配，这里的2是怎么来的？

这里的2是根据前后缀的相同程度来得到的，以`ABCDABD`为例：

```
－　"A"的前缀和后缀都为空集，共有元素的长度为0；

－　"AB"的前缀为[A]，后缀为[B]，共有元素的长度为0；

－　"ABC"的前缀为[A, AB]，后缀为[BC, C]，共有元素的长度0；

－　"ABCD"的前缀为[A, AB, ABC]，后缀为[BCD, CD, D]，共有元素的长度为0；

－　"ABCDA"的前缀为[A, AB, ABC, ABCD]，后缀为[BCDA, CDA, DA, A]，共有元素为"A"，长度为1；

－　"ABCDAB"的前缀为[A, AB, ABC, ABCD, ABCDA]，后缀为[BCDAB, CDAB, DAB, AB, B]，共有元素为"AB"，长度为2；

－　"ABCDABD"的前缀为[A, AB, ABC, ABCD, ABCDA, ABCDAB]，后缀为[BCDABD, CDABD, DABD, ABD, BD, D]，共有元素的长度为0。
```

可以看到"ABCDAB"的前后缀匹配长度是2也就是AB。知道了这个，接下来就是根据p求各个子串的前后缀的匹配长度了。

这里使用归纳法来求，用next[j]表示需要回退的索引，假设我们已知next[0...j]，求next[j+1]。

现在我们假设next[j]=k，现在求next[j+1]：

1> 如果p[j]==p[k], 此时next[j+1]=k+1;

2> 如果p[j]!=p[k], 那么只能去看next[k]这段有没有相同的前后缀，所以这个时候需要更新k，k=next[k]; 此后就会重复1，2步骤。

初始设置next[0]=-1, k=-1, j=0。当next[j]==-1的时候，表示p需要从0开始查找。

---

有了next数组，当s[i]!=p[j]时，更新j=next[j]。但是如果p[j]==p[next[j]]，这个时候与p[i]的比较依然是会失败的。

所以针对这种情况，需要进行一个优化。

在之前的第一步中,当p[j]==p[k], 此时设置了next[j+1]=k+1。此时设置的next[j+1]就是当s[i]和p[j]比较失败的时候，需要返回的索引,其实也就是k+1。
所以前面说的p[j]==p[next[j]]，其实就是比较p[j+1]==p[k+1]。如果相等，只能丢弃当前的的前后缀匹配(k+1), 返回对应k+1处的前后缀匹配了，也就是next[++j] = next[++k]。

到此为止，next数组求完，有了next数组，查找其实就很简单了，下面笔者给出参考代码：

{% highlight cpp %}
 int strStr(string haystack, string needle) {
	if (needle.empty()) { return 0; }

	auto get_next = [](string p) {
		vector<int> next(p.size(), 0); next[0] = -1;
		int k = -1, j = 0;

		// [0...j] -> [j+1]
		while (j < p.size()-1) {
			if (k == -1 || p[j] == p[k]) {
				if (p[j+1] == p[k+1]) {
					next[++j] = next[++k];
				} else {
					next[++j] = ++k;
				}
			} else {
				k = next[k];
			}
		}

		return next;
	};

	int i = 0, j = 0;
	int l1 = haystack.size(), l2 = needle.size();
	vector<int> next = get_next(needle);

	while (i < l1 && j < l2) {
		if (j == -1 || haystack[i] == needle[j]) {
			i++; j++;
		} else {
			j = next[j];
		}
	}

	if (j == l2) {
		return i - l2;
	}

	return -1;
}
{% endhighlight %}

## dfa

KMP还有一种更加巧妙的实现方式，使用确定有限状态机，也就是dfa。状态机本身有多个状态，通过不同的输入可以进行状态间的转移，如下图所示:

![string_algorithm_1]({{site.url}}/assets/images/blog/string_algorithm_1.png)

我们要做的就是构建一个类似上图的状态机，输入也就是s中各个字符，当s中某段就是p那么状态一直往前直到结束，也就匹配结束。

我们实现用的是二维数组来存状态转移的信息，下面依然给出"ABCDABD"的二维数组形式的状态机:

```
		0	1	2	3	4	5	6
		A	B	C	D	A	B	D
------+---+---+---+---+---+---+---+
A	  | 1 |   |   |   | 5 |   |   |
------+---+---+---+---+---+---+---+
B	  |   | 2 |   |   |   | 6 |   |
------+---+---+---+---+---+---+---+
C	  |   |   | 3 |   |   |   |   |
------+---+---+---+---+---+---+---+
D	  |   |   |   | 4 |   |   | 7 | 
------+---+---+---+---+---+---+---+      
```

上图就是一个从前往后直到匹配完成的状态转移过程，7就是代表匹配结束了。

```
		0	1	2	3	4	5	6
		A	B	C	D	A	B	D
------+---+---+---+---+---+---+---+
A	  | 1 |   |   |   |   |   |   |
------+---+---+---+---+---+---+---+
B	  | 0 |   |   |   |   |   |   |
------+---+---+---+---+---+---+---+
C	  | 0 |   |   |   |   |   |   |
------+---+---+---+---+---+---+---+
D	  | 0 |   |   |   |   |   |   | 
------+---+---+---+---+---+---+---+
```

首先初始状态0，只有当输入是A才会进入当下一状态1，其他输入依然会停留在状态0。其实就是s[i]!=p[j],这个时候，i往前移动，j依然还是0。

```
		0	1	2	3	4	5	6
		A	B	C	D	A	B	D
------+---+---+---+---+---+---+---+
A	  | 1 | 1 |   |   |   |   |   |
------+---+---+---+---+---+---+---+
B	  | 0 | 2 |   |   |   |   |   |
------+---+---+---+---+---+---+---+
C	  | 0 | 0 |   |   |   |   |   |
------+---+---+---+---+---+---+---+
D	  | 0 | 0 |   |   |   |   |   | 
------+---+---+---+---+---+---+---+
```

现在到了状态1，当输入是B进入状态2没有疑问。下面就是来看当状态1时，输入非B的时候，状态如何转移。

先看状态1，输入是A，其实结合字符串的比较想就会很清楚了，现在状态是1那么证明上一个比较的字符是A，不然不可能位于状态1。所以在状态1输入A，其实也就是s="AA...."，所以依然停留在状态1即可。也就是说接下来的比较依然是s[x] 和 p[1] 进行比，因为前面一个A是匹配的。

在看状态1，输入是C，也就是此时s="AC...."，所以接下来比较就是s[x] 和 p[0]进行比较，对应到状态也就是回到了状态0，需要重新开始一次比较。

现在在看，第二列其实就是第一列的复制，除了2的那个坐标。也就是说在求接下来的列中的时候，会使用到前面列的信息，下面接着看。

```
		0	1	2	3	4	5	6
		A	B	C	D	A	B	D
------+---+---+---+---+---+---+---+
A	  | 1 | 1 | 1 |   |   |   |   |
------+---+---+---+---+---+---+---+
B	  | 0 | 2 | 0 |   |   |   |   |
------+---+---+---+---+---+---+---+
C	  | 0 | 0 | 3 |   |   |   |   |
------+---+---+---+---+---+---+---+
D	  | 0 | 0 | 0 |   |   |   |   | 
------+---+---+---+---+---+---+---+
```

现在到了状态2，当输入是C进入状态3。

当输入为A时，此时s="ABA...."，接下来比较是s[x] 和 p[1] ，因为p的第一位就是1，所以从第二位开始比就好，所以状态依然是1。
当输入为B，D时，状态则回到0。

此时第三列依然是第1列的复制，当然除了3坐标。

```
		0	1	2	3	4	5	6
		A	B	C	D	A	B	D
------+---+---+---+---+---+---+---+
A	  | 1 | 1 | 1 | 1 |   |   |   |
------+---+---+---+---+---+---+---+
B	  | 0 | 2 | 0 | 0 |   |   |   |
------+---+---+---+---+---+---+---+
C	  | 0 | 0 | 3 | 0 |   |   |   |
------+---+---+---+---+---+---+---+
D	  | 0 | 0 | 0 | 4 |   |   |   | 
------+---+---+---+---+---+---+---+
```

到了状态3，不多说，看下一个状态。

```
		0	1	2	3	4	5	6
		A	B	C	D	A	B	D
------+---+---+---+---+---+---+---+
A	  | 1 | 1 | 1 | 1 | 5 |   |   |
------+---+---+---+---+---+---+---+
B	  | 0 | 2 | 0 | 0 | 0 |   |   |
------+---+---+---+---+---+---+---+
C	  | 0 | 0 | 3 | 0 | 0 |   |   |
------+---+---+---+---+---+---+---+
D	  | 0 | 0 | 0 | 4 | 0 |   |   | 
------+---+---+---+---+---+---+---+
```

到了状态4，第5列依然还是第1列的复制。

但是此时需要注意的时，接下来求第6列的时候，就不是从第0列复制了，而是从第2列复制，也就是状态1。

为什么呢？因为当到了状态5的时候，p前面是"ABCDA"，"ABCDA" 在前面说到它的前后缀匹配是1，所以在求第6列的时候，需要利用前后缀匹配。

```
		0	1	2	3	4	5	6
		A	B	C	D	A	B	D
------+---+---+---+---+---+---+---+
A	  | 1 | 1 | 1 | 1 | 5 | 1 |   |
------+---+---+---+---+---+---+---+
B	  | 0 | 2 | 0 | 0 | 0 | 6 |   |
------+---+---+---+---+---+---+---+
C	  | 0 | 0 | 3 | 0 | 0 | 0 |   |
------+---+---+---+---+---+---+---+
D	  | 0 | 0 | 0 | 4 | 0 | 0 |   | 
------+---+---+---+---+---+---+---+
```

到了状态5，从状态1也就是第二列复制。

此时p前面是"ABCDAB", 前后缀匹配是2，所以继续更新复制的列为状态2所在列。

```
		0	1	2	3	4	5	6
		A	B	C	D	A	B	D
------+---+---+---+---+---+---+---+
A	  | 1 | 1 | 1 | 1 | 5 | 1 | 1 |
------+---+---+---+---+---+---+---+
B	  | 0 | 2 | 0 | 0 | 0 | 6 | 0 |
------+---+---+---+---+---+---+---+
C	  | 0 | 0 | 3 | 0 | 0 | 0 | 3 |
------+---+---+---+---+---+---+---+
D	  | 0 | 0 | 0 | 4 | 0 | 0 | 7 | 
------+---+---+---+---+---+---+---+
```

到了状态6，从状态2也就是第3列进行复制。

看一个例子，当状态5，输入C，此时s="ABCDABC...."，所以下一次的比较是s[x] 和 p[3]即可，因为前面ABC和p一致，这也就是使用列前后缀的信息。

现在对dfa的计算有了一个大致的了解，下面最后一个任务就是代码实现上述过程，下面笔者给出参考代码:
               
{% highlight cpp %}
vector<vector<int>> get_dfa(string p) {
    vector<vector<int>> dfa(256, vector<int>(p.size(), 0));
    dfa[p[0]][0] = 1;

    for (int j = 1, state = 0; j < p.size(); ++j) {
        for (int i = 0; i < 256; ++i) {
            dfa[i][j] = dfa[i][state]; // 从前一列进行复制
        }
        dfa[p[j]][j] = j + 1; // 更新每次匹配成功的那个状态
        state = dfa[p[j]][state]; // 尝试更新复制的列
    }

    return dfa;
}
{% endhighlight %}

有了dfa，kmp就很简单了，而且此时的search是不用进行比较了，直接更新j即可，参考代码如下：

{% highlight cpp %}
int strStr_dfa(string haystack, string needle) {
    if (needle.empty()) { return 0; }

    auto get_dfa = [](string p) {
        vector<vector<int>> dfa(256, vector<int>(p.size(), 0));
        dfa[p[0]][0] = 1;

        int state = 0;
        for (int i = 1; i < p.size(); ++i) {
            for (int j = 0; j < 256; ++j) {
                dfa[j][i] = dfa[j][state];
            }	
            dfa[p[i]][i] = i+1;
            state = dfa[p[i]][state];
        }

        return dfa;
    };

    int i = 0, j = 0;
    int l1 = haystack.size(), l2 = needle.size();
    auto dfa = get_dfa(needle);

    for (; i < l1 && j < l2; ++i) {
        j = dfa[haystack[i]][j];
    }
    if (j == l2) { return i - l2; }
    return -1;
}
{% endhighlight %}

# Trie

Trie也就是所谓的字典树，一般用在搜索提示和分词。

下面给出一个Trie的简单实现:

{% highlight cpp%}
struct TrieNode {
	bool is_end = false;
	array<TrieNode*, 26> children = { nullptr };

	~TrieNode() {
		for (auto child: children) {
			delete child;
			child = nullptr;
		}
	}
};

class Trie {
	TrieNode* root;
public:
    /** Initialize your data structure here. */
    Trie() {
		root = new TrieNode();
    }

	~Trie() {
		delete root;
		root = nullptr;
	}
    
    /** Inserts a word into the trie. */
    void insert(string word) {
       	auto node = root;
		for (auto ch: word) {
			int idx = ch - 'a';
			if (node->children[idx] == nullptr) {
				node->children[idx] = new TrieNode();
			} 
			node = node->children[idx];
		}
		node->is_end = true;
    }
    
    /** Returns if the word is in the trie. */
    bool search(string word) {
       	auto node = root;
		for (auto ch : word) {
			int idx = ch - 'a';
			if (node->children[idx] == nullptr) {
				return false;
			}
			node = node->children[idx];
		}
		return node->is_end; // case insert("apple"); search("app")
    }
}
{% endhighlight %}

# AC Automaton

当给定一个字符串和多个子串进行查找时。这个时候可以对多个子串依次进行KMP查找，但是当子串较多时，此时效率并不高。

AC自动机就是来解决这类问题的，AC自动机首先时一颗Trie，其次使用了KMP的思想，使用fail指针来尝试不匹配的指向。

# Suffix Array


# References

## KMP

[https://www.cnblogs.com/tangzhengyue/p/4315393.html](https://www.cnblogs.com/tangzhengyue/p/4315393.html)

[https://blog.csdn.net/v_july_v/article/details/7041827](https://blog.csdn.net/v_july_v/article/details/7041827)

[http://www.ruanyifeng.com/blog/2013/05/Knuth%E2%80%93Morris%E2%80%93Pratt_algorithm.html](http://www.ruanyifeng.com/blog/2013/05/Knuth%E2%80%93Morris%E2%80%93Pratt_algorithm.html)

[https://judes.me/tech/2016/04/10/kmp.html](https://judes.me/tech/2016/04/10/kmp.html)

[https://blog.csdn.net/congduan/article/details/45459963](https://blog.csdn.net/congduan/article/details/45459963)

## AC Automaton

[https://www.cnblogs.com/nullzx/p/7499397.html](https://www.cnblogs.com/nullzx/p/7499397.html)

[https://segmentfault.com/a/1190000000484127](https://segmentfault.com/a/1190000000484127)

## Suffix Array

[https://blog.bill.moe/suffix-array-notes/](https://blog.bill.moe/suffix-array-notes/)

[https://www.cnblogs.com/jinkun113/p/4743694.html](https://www.cnblogs.com/jinkun113/p/4743694.html)

[https://oi.men.ci/suffix-array-notes/](https://oi.men.ci/suffix-array-notes/)
