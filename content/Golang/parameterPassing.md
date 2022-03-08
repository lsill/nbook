---
title: "参数传递语法糖注意"
date: 2022-03-08T16:47:48+08:00
draft: false
---



​	`golang` 支持函数中任意参数的传递，写法如下

```go
func PrintInput(str ...string) {
	fmt.Println(fmt.Sprintf("%T", str))
}
```

运行上面函数这里输出

```
[]string
```

传递的参数本质是一个数组，所以我们在传参的时候某些情况需要注意参数的实际数量。

单独做一个长度的判定（也是看需求）。

例子：

```go
type List struct {
	elements []interface{}
	size     int
}

type Heap struct {
	list       *List
}

func (l *List) Add(values ...interface{}) {
	l.growBy(len(values))	// 扩容
	for _, value := range values {
		l.elements[l.size] = value
		l.size++
	}
}

func (heap *Heap) Push(values ...interface{}) {
	if len(values) == 1 {
		heap.list.Add(values[0])	// 这里如果不做索引判断，传入的参数会是[]interface，后面的类型推断（断言）将会报错
		heap.bubbleUp()
	} else {
		for _, value := range values {
			heap.list.Add(value)	
		}
		size := heap.list.Size()/2 + 1
		for i := size; i >= 0; i-- {
			heap.bubbleDownIndex(i)
		}
	}
}
```

