---
layout: post
title: "红黑树"
description: ""
category: algorithm
tags: [算法, 红黑树]
---
{% include JB/setup %}

## 概要

红黑树是满足下列约束条件的二叉搜索树：

  1. 每个节点不是红色，就是黑色
  2. 根节点为黑色
  3. 红色节点的子节点必须为黑色
  4. 任一节点至叶子节点的任何路径，所含之黑节点数必须相等


这些约束确保了红黑树的关键特性：从根到叶子的最长的可能路径不多于最短的可能路径的两倍长。所以这棵树大致是平衡的。

## 插入操作

我们首先以二叉查找树的方法增加节点并标记它为红色。
（如果设为黑色，就会导致根到叶子的路径中有一条路径，多一个额外的黑节点，这个是很难调整的。
但是设为红色节点后，可能会导致出现两个连续红色节点的冲突，那么可以通过颜色调换（color flips）和树旋转来调整。）
下面要进行什么操作取决于其他临近节点的颜色。同人类的家族树中一样，我们将使用术语叔父节点来指一个节点的父节点的兄弟节点。
注意：

  1. 性质1总是保持着。
  2. 性质3只在增加红色节点，重绘黑色节点为红色，或做旋转时受到威胁。
  3. 性质4只在增加黑色节点，重绘红色节点为黑色，或做旋转时受到威胁。

在接下来的讨论中，我们将要插入的节点标为N，N的父节点标为P，N的祖父节点标为G，N的叔父节点标为U。

我们使用C++代码来展示算法过程。如：

``` c++
inline Node* grandParent(Node *n) const
{
	return n->parent->parent;
}

inline Node* uncle(Node *n) const
{
	if (grandParent(n)->left == n->parent)
		return grandParent(n)->right;
	else
		return grandParent(n)->left;
}

inline Node* sibling(Node *n) const
{
	if (n->parent->left == n)
		return n->parent->right;
	else
		return n->parent->left;
}
```

### 情形1

新节点N位于树的根上，没有父节点。在这种情形下，我们把它重绘为黑色以满足性质2。
因为它在每个路径上对黑节点数目增加一，性质4符合。

``` c++
void insertCase1(NodeType *n)
{
	if (n->parent == NULL)
		n->color = RBTree_Black;
	else
		insertCase2(n);
}
```

### 情形2

![](/images/algorithm/rbtree/insert-case2.png)

新节点的父节点P是黑色，所以性质3没有失效（新节点是红色的），性质4也未受到威胁。
在这种情形下，树仍是有效的。

``` c++
void insertCase2(NodeType *n)
{
	if (n->parent->color == RBTree_Black)
		return;
	else
		insertCase3(n);
}
```

### 情形3

![](/images/algorithm/rbtree/insert-case3.png)

如果父节点P和叔父节点U二者都是红色，
（此时新插入节点N做为P的左子节点或右子节点都属于情形3，这里右图仅显示N做为P左子的情形）
则我们可以将它们两个重绘为黑色并重绘祖父节点G为红色（用来保持性质5）。
现在我们的新节点N有了一个黑色的父节点P。
因为通过父节点P或叔父节点U的任何路径都必定通过祖父节点G，在这些路径上的黑节点数目没有改变。
但是，红色的祖父节点G可能是根节点，这就违反了性质2，也有可能祖父节点G的父节点是红色的，这就违反了性质3。
为了解决这个问题，我们在祖父节点G上递归地进行情形1的整个过程。（把G当成是新加入的节点进行各种情形的检查）

``` c++
void insertCase3(NodeType *n)
{
	if (uncle(n) && uncle(n)->color == RBTree_Red)
	{
		grandParent(n)->color = RBTree_Red;
		uncle(n)->color = RBTree_Black;
		n->parent->color = RBTree_Black;
		insertCase1(grandParent(n));
	}
	else
	{
		insertCase4(n);
	}
}
```

### 情形4

![](/images/algorithm/rbtree/insert-case4.png)

父节点P是红色而叔父节点U是黑色或缺少，并且新节点N是其父节点P的右子节点而父节点P又是其父节点的左子节点。
在这种情形下，我们进行一次左旋转调换新节点和其父节点的角色；
接着，我们按情形5处理以前的父节点P以解决仍然失效的性质3。
注意这个改变会导致某些路径通过它们以前不通过的新节点N或P，但由于这两个节点都是红色的，所以性质4仍有效。

``` c++
void insertCase4(NodeType *n)
{
	NodeType *x = n;
	if (grandParent(n)->left == n->parent && n->parent->right == n)
	{
		rotateLeft(n->parent);
		x = n->left;
	}
	else if (grandParent(n)->right == n->parent && n->parent->left == n)
	{
		rotateRight(n->parent);
		x = n->right;
	}
	insertCase5(x);
}
```

### 情形5

![](/images/algorithm/rbtree/insert-case5.png)

父节点P是红色而叔父节点U是黑色或缺少，新节点N是其父节点的左子节点，而父节点P又是其父节点G的左子节点。
在这种情形下，我们进行针对祖父节点G的一次右旋转；
在旋转产生的树中，以前的父节点P现在是新节点N和以前的祖父节点G的父节点。
我们知道以前的祖父节点G是黑色，否则父节点P就不可能是红色。
我们切换以前的父节点P和祖父节点G的颜色，得到的树满足性质3。
性质4也仍然保持满足，因为通过这三个节点中任何一个的所有路径以前都通过祖父节点G，现在它们都通过以前的父节点P。
在各自的情形下，这都是三个节点中唯一的黑色节点。

``` c++
void insertCase5(NodeType *n)
{
	grandParent(n)->color = RBTree_Red;
	n->parent->color = RBTree_Black;

	if (grandParent(n)->left == n->parent)
	{
		rotateRight(grandParent(n));
	}
	else
	{
		rotateLeft(grandParent(n));
	}
}
```

## 删除操作

如果需要删除的节点有两个儿子，那么问题可以被转化成删除另一个只有一个儿子的节点的问题。
对于二叉查找树，在删除带有两个非叶子儿子的节点的时候，
我们找到要么在它的左子树中的最大元素、要么在它的右子树中的最小元素，并把它的值转移到要删除的节点中。
我们接着删除我们从中复制出值的那个节点，它必定有少于两个非叶子的儿子。
因为只是复制了一个值，不违反任何性质，这就把问题简化为如何删除最多有一个儿子的节点的问题。
它不关心这个节点是最初要删除的节点还是我们从中复制出值的那个节点。

在本文余下的部分中，我们只需要讨论删除只有一个儿子的节点。
如果我们删除一个红色节点，它的父亲和儿子一定是黑色的。
所以我们可以简单的用它的黑色儿子替换它，并不会破坏性质2和性质3。
通过被删除节点的所有路径只是少了一个红色节点，这样可以继续保证性质4。
另一种简单情况是在被删除节点是黑色而它的儿子是红色的时候。
如果只是去除这个黑色节点，用它的红色儿子顶替上来的话，会破坏性质4，
但是如果我们重绘它的儿子为黑色，则曾经通过它的所有路径将通过它的黑色儿子，这样可以继续保持性质4。

需要进一步讨论的是在要删除的节点和它的儿子二者都是黑色的时候，这是一种复杂的情况。
我们首先把要删除的节点替换为它的儿子。出于方便，称呼这个儿子为N（在新的位置上），称呼它的兄弟（它父亲的另一个儿子）为S。
在下面的示意图中，我们还是使用P称呼N的父亲，L称呼S的左儿子，R称呼S的右儿子。

### 情形1

N是新的根。在这种情形下，我们就做完了。我们从所有路径去除了一个黑色节点，而新根是黑色的，所以性质都保持着。

``` c++
void deleteCase1(NodeType *n)
{
	if (n != mRoot)
		deleteCase2(n);
}
```

### 情形2

![](/images/algorithm/rbtree/delete-case2.png)

S是红色。在这种情形下我们在N的父亲上做左旋转，把红色兄弟转换成N的祖父，我们接着对调N的父亲和祖父的颜色。
完成这两个操作后，尽管所有路径上黑色节点的数目没有改变，
但现在N有了一个黑色的兄弟和一个红色的父亲（它的新兄弟是黑色因为它是红色S的一个儿子），
所以我们可以接下去按情形4、情形5或情形6来处理。

``` c++
void deleteCase2(NodeType *n)
{
	NodeType* s = sibling(n);
	if (s->color == RBTree_Red)
	{
		n->parent->color = RBTree_Red;
		s->color = RBTree_Black;
		if (s->parent->left == s)
			rotateRight(n->parent);
		else
			rotateLeft(n->parent);
	}
	deleteCase3(n);
}
```

### 情形3

![](/images/algorithm/rbtree/delete-case3.png)

N的父亲、S和S的儿子都是黑色的。在这种情形下，我们简单的重绘S为红色。
结果是通过S的所有路径，它们就是以前不通过N的那些路径，都少了一个黑色节点。
因为删除N的初始的父亲使通过N的所有路径少了一个黑色节点，这使事情都平衡了起来。
但是，通过P的所有路径现在比不通过P的路径少了一个黑色节点，所以仍然违反性质4。
要修正这个问题，我们要从情形1开始，在P上做重新平衡处理。

``` c++
void deleteCase3(NodeType *n)
{
	NodeType *s = sibling(n);
	if (s->color == RBTree_Black &&
		(s->left == NULL || s->left->color == RBTree_Black) &&
		(s->right == NULL || s->right->color == RBTree_Black) &&
		n->parent->color == RBTree_Black)
	{
		s->color = RBTree_Red;
		deleteCase1(n->parent);
	}
	else
	{
		deleteCase4(n);
	}
}
```

### 情形4

![](/images/algorithm/rbtree/delete-case4.png)

S和S的儿子都是黑色，但是N的父亲是红色。在这种情形下，我们简单的交换N的兄弟和父亲的颜色。
这不影响不通过N的路径的黑色节点的数目，但是它在通过N的路径上对黑色节点数目增加了一，添补了在这些路径上删除的黑色节点。

``` c++
void deleteCase4(NodeType *n)
{
	NodeType *s = sibling(n);
	if (s->color == RBTree_Black &&
		(s->left == NULL || s->left->color == RBTree_Black) &&
		(s->right == NULL || s->right->color == RBTree_Black) &&
		n->parent->color == RBTree_Red)
	{
		n->parent->color = RBTree_Black;
		s->color = RBTree_Red;
	}
	else
	{
		deleteCase5(n);
	}
}
```

### 情形5

![](/images/algorithm/rbtree/delete-case5.png)

S是黑色，S的左儿子是红色，S的右儿子是黑色，而N是它父亲的左儿子。
在这种情形下我们在S上做右旋转，这样S的左儿子成为S的父亲和N的新兄弟。
我们接着交换S和它的新父亲的颜色。所有路径仍有同样数目的黑色节点，
但是现在N有了一个黑色兄弟，他的右儿子是红色的，所以我们进入了情形6。
N和它的父亲都不受这个变换的影响。

``` c++
void deleteCase5(NodeType *n)
{
	NodeType *s = sibling(n);
	if ((n->parent->left == n) &&
		(s->color == RBTree_Black &&
		 (s->right == NULL || s->right->color == RBTree_Black) &&
		 (s->left && s->left->color == RBTree_Red)))
	{
		s->color = RBTree_Red;
		s->left->color = RBTree_Black;
		rotateRight(s);
	}
	else if ((n->parent->right == n) &&
			 (s->color == RBTree_Black &&
			  (s->left == NULL || s->left->color == RBTree_Black) &&
			  (s->right && s->right->color == RBTree_Red)))
	{
		s->color = RBTree_Red;
		s->right->color = RBTree_Black;
		rotateLeft(s);
	}
	deleteCase6(n);
}
```

### 情形6

![](/images/algorithm/rbtree/delete-case6.png)

S是黑色，S的右儿子是红色，而N是它父亲的左儿子。
在这种情形下我们在N的父亲上做左旋转，这样S成为N的父亲P和S的右儿子的父亲。
我们接着交换N的父亲和S的颜色，并使S的右儿子为黑色。
子树在它的根上的仍是同样的颜色，所以性质2没有被违反。
但是，N现在增加了一个黑色祖先：要么N的父亲变成黑色，要么它是黑色而S被增加为一个黑色祖父。
所以，通过N的路径都增加了一个黑色节点。

``` c++
void deleteCase6(NodeType *n)
{
	NodeType *s = sibling(n);
	s->color = n->parent->color;
	n->parent->color = RBTree_Black;

	if (n->parent->left == n)
	{
		s->right->color = RBTree_Black;
		rotateLeft(n->parent);
	}
	else
	{
		s->left->color = RBTree_Black;
		rotateRight(n->parent);
	}
}
```

*完整的代码参见： [红黑树示例](https://github.com/ruleless/stl_test/blob/master/RBTree.h)*
