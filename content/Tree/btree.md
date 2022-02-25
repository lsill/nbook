---
title: "bTree 分析"
date: 2022-02-15T16:57:48+08:00
draft: true
---



## 此文是笔记

### 1.btree定义

[定义可以查看维基百科](https://zh.wikipedia.org/wiki/B%E6%A0%91)



### 2. 实现拆分

![btree](/Users/lsill/gitclone/nbook/static/images/tree/btree.png)

### 3.实现

[作者 实现地址 github传送 ](https://github.com/emirpasic/gods/tree/master/trees/btree)

#### 1. 节点定义

```golang
// Node 树中的一个元素
type Node struct {
	Parent   *Node	// 父节点
	Entries  []*Entry	// 包含的键值对
	Children []*Node	// 子节点
}
```



##### 2. 树的定义

```go
 type Tree struct {
	Root       *Node // 根节点
	Comparator utils.Comparator	// 比较函数 用来给树排序
	size       int	// 节点数量
	m          int	// 叶节点数 ？？？
 }
```



#### 3. 比较函数的实现

```go
type Comparator func(a, b interface{}) int

func IntComparator(a, b interface{}) int {
	aAsserted := a.(int)
	bAsserted := b.(int)
	switch {
	case aAsserted > bAsserted:
		return 1
	case aAsserted < bAsserted:
		return -1
	default:
		return 0
	}
}

func StringComparator(a, b interface{}) int {
	s1 := a.(string)
	s2 := b.(string)
	min := len(s1)
	if len(s1) < len(s2) {
		min = len(s1)
	}
	diff := 0
	for i := 0; i < min && diff == 0; i++ {
		diff = int(s1[i]) - int(s2[i])
	}
	if diff == 0 {
		diff = len(s1) - len(s2)
	}
	if diff < 0 {
		return -1
	}
	if diff > 0 {
		return 1
	}
	return 0
}
```



#### 4. 创建树

```go
func NewWith(order int, comparator utils.Comparator) *Tree {
	if order < 3 {
		panic("Invalid order, should be at least 3")	// 最小为二叉树
	}
	return &Tree{
		Comparator: comparator,
		m:          order,
	}
}

func NewWithIntComparator(order int) *Tree {
	return NewWith(order, utils.IntComparator)
}

func NewWithStringComparator(order int) *Tree {
	return NewWith(order, utils.StringComparator)
}
```



#### 5.查找节点

```go
func (t *Tree) search(node *Node, key interface{}) (index int, found bool) {
	low, high := 0, len(node.Entries)-1	// 二分查找
	var mid int
	for low <= high {
		mid = (high + low) / 2
		compare := t.Comparator(key, node.Entries[mid].Key)
		switch {
		case compare > 0:
			low = mid + 1
		case compare < 0:
			high = mid - 1
		case compare == 0:
			return mid, true
		}
	}
	return low, false
}
```



#### 6. 插入节点

```go
// Put bTree中插入新的节点
func (t *Tree) Put(key interface{}, value interface{}) {
	entry := &Entry{	// 创建值
		Key:   key,
		Value: value,
	}
	if t.Root == nil {	// 根节点为空 创建根节点
		t.Root = &Node{
			Parent:   nil,
			Entries:  []*Entry{entry},
			Children: []*Node{},
		}
		t.size++
		return
	}
	if t.insert(t.Root, entry) {
		t.size++
	}
}

// insert 实际的插入操作
func (t *Tree) insert(node *Node, entry *Entry) bool {
  if t.isLeaf(node) {	// 判断是否是叶节点（len(node.Children) == 0）
		return t.insertIntoLeaf(node, entry)
	}
	return t.insertIntoInternal(node, entry)
}

// insertIntoLeaf 像叶节点插入
func (t *Tree) insertIntoLeaf(node *Node, entry *Entry) (inserted bool) {
	insertPosition, found := t.search(node, entry.Key)
	if found {
		node.Entries[insertPosition] = entry	// 更新节点值
		return false
	}
	// Insert entry's key in the middle of the node
	node.Entries = append(node.Entries, nil)
	copy(node.Entries[insertPosition+1:], node.Entries[insertPosition:])
	node.Entries[insertPosition] = entry
	t.split(node)
	return true
}

// insertIntoInternal 像内部节点插入
func (t *Tree) insertIntoInternal(node *Node, entry *Entry) (inserted bool) {
	insertPosition, found := t.search(node, entry.Key)
	if found {
		node.Entries[insertPosition] = entry
		return false
	}
	return t.insert(node.Children[insertPosition], entry)
}
```



#### 7. 分割节点

```go
func (t *Tree) split(node *Node) {
	if !t.shouldSplit(node) {
		return
	}
	if node == t.Root {
		t.splitRoot()
		return
	}
	t.splitNonRoot(node)
}

// splitRoot 分割根节点
func (t *Tree) splitRoot() {
	middle := t.middle()
	left := &Node{Entries: append([]*Entry(nil), t.Root.Entries[:middle]...)}
	right := &Node{Entries: append([]*Entry(nil), t.Root.Entries[middle+1:]...)}

	// 将子节点从要拆分的节点中移动为 左右 节点
	if !t.isLeaf(t.Root) {
		left.Children = append([]*Node(nil), t.Root.Children[:middle+1]...)
		right.Children = append([]*Node(nil), t.Root.Children[middle+1:]...)
		setParent(left.Children, left)
		setParent(right.Children, right)
	}

	// Root is a node with one entry and two children (left and right)
	newRoot := &Node{
		Entries:  []*Entry{t.Root.Entries[middle]},
		Children: []*Node{left, right},
	}
	left.Parent = newRoot
	right.Parent = newRoot
	t.Root = newRoot
}

// splitNonRoot 分割非根节点
func (t *Tree) splitNonRoot(node *Node) {
	middle := t.middle()
	parent := node.Parent

	left := &Node{Entries: append([]*Entry(nil), node.Entries[:middle]...), Parent: parent}
	right := &Node{Entries: append([]*Entry(nil), node.Entries[middle+1:]...), Parent: parent}

	// 将子节点从要拆分的节点中移动为 左右 节点
	if !t.isLeaf(node) {
		left.Children = append([]*Node(nil), node.Children[:middle+1]...)
		right.Children = append([]*Node(nil), node.Children[middle+1:]...)
		setParent(left.Children, left)
		setParent(right.Children, right)
	}

	insertPosition, _ := t.search(parent, node.Entries[middle].Key)

	// Insert middle key into parent
	parent.Entries = append(parent.Entries, nil)
	copy(parent.Entries[insertPosition+1:], parent.Entries[insertPosition:])
	parent.Entries[insertPosition] = node.Entries[middle]

	// Set child left of inserted key in parent to th created left node
	parent.Children[insertPosition] = left

	// Set child right of inserted key in parent to the created right node
	parent.Children = append(parent.Children, nil)
	copy(parent.Children[insertPosition+2:], parent.Children[insertPosition+1:])
	parent.Children[insertPosition+1] = right

	t.split(parent)
}
```









