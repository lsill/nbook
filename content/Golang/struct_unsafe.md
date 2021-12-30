---
title: "struct的一些操作"
date: 2021-12-30T12:37:48+08:00
draft: false
---



此文为笔记

[原文](https://mp.weixin.qq.com/s/A4m1xlFwh9pD0qy3p7ItSA)

## NoCopy

test1.go

```go
package main

import (
 "sync"
)

func test(wg sync.WaitGroup) {
 defer wg.Done()
 wg.Add(1)
}

func main() {
 var wg sync.WaitGroup
 wg.Add(1)
 go test(wg)
 wg.Wait()
}
```

上面的代码可以通过编译，不过在运行的时候会报错。我在写代码的时候很少用go vet 进行语法检查，以后需要注意下用。

输入go vet test1.go

![struct_nocopy](E:\github\nbook\static\images\golang\struct_nocopy.PNG)

上面写着因为sync.WaitGroup包含 sync.noCopy,所以无法进行锁的值拷贝。

所以 如果传递值改为传递锁的指针的话就可以通过go vet检查。

sync.WaitGruop实现

```go
type WaitGroup struct {
	noCopy noCopy

	// 64-bit value: high 32 bits are counter, low 32 bits are waiter count.
	// 64-bit atomic operations require 64-bit alignment, but 32-bit
	// compilers only guarantee that 64-bit fields are 32-bit aligned.
	// For this reason on 32 bit architectures we need to check in state()
	// if state1 is aligned or not, and dynamically "swap" the field order if
	// needed.
	state1 uint64
	state2 uint32
}
```

其中包含noCopy，而noCopy的实现

```go
type noCopy struct{}

// Lock is a no-op used by -copylocks checker from `go vet`.
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```

这个noCopy设计就是为了在go vet的时候不通过，换句话是拒绝值拷贝的一种方式，我们可以尝试换种方式写看看能不能通过go vet

```go
func test(t Test1) {
	fmt.Println(t.a)
}

func main() {
	t := Test1{
		a: 1,
	}
	test(t)
}

type Test1 struct {
	noCopy
	a int
}

type noCopy struct{}

// Lock is a no-op used by -copylocks checker from `go vet`.
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```

这么写运行是没有问题，但在go vet的时候就会输出:

![struct_nocopy_1](E:\github\nbook\static\images\golang\struct_nocopy_1.PNG)

好像只能在检查时候才能做到不允许拷贝的操作。

grpc是这么实现DoNotCopy的

```go
type DoNotCopy [0]sync.Mutex
```

用这种方式在实现进行操作

```go
func test(t Test1) {
	fmt.Println(t.a)
}

func main() {
	t := Test1{
		a: 1,
	}
	test(t)
}

type DoNotCopy [0]sync.Mutex

type Test1 struct {
	DoNotCopy
	a int
}
```

一样的可以编译通过并且运行，但就是无法通过go vet检测

输出：

![struct_nocopy_2](E:\github\nbook\static\images\golang\struct_nocopy_2.PNG)

**作用**：用这个的话，可以在go vet期间发现值拷贝导致没有正确修改传递的结构体的值。在实际项目中有很多传参，不一定什么时候就传错了，可以记下。而且**空结构体不占据任何空间，不会对性能有任何影响。**



## DoNotCompare

golang中**slice、map、function都是引用类型不可比较的，只能判断是否为空。**但值类型可以相互比较，**strcut之间可以互相比较，只有所有字段全部 comparable 的(不限大小写是否导出)，那么结构体才可以比较。同时只比较 non-blank 的字段。**

如果结构体中有non-blank的字段的话会输出什么

```golang
type T struct {
    name string
    age int
    _ float64
}
func main() {
   x := [...]float64{1.1, 2, 3.14}
   fmt.Println(x == [...]float64{1.1, 2, 3.14}) // true
   y := [1]T{{"foo", 1, 0}}
   fmt.Println(y == [1]T{{"foo", 1, 1}}) // true
}
```

两个不同的结构体最终会相等，这不够明确，如果用struct作为map的key值得话，那就不满足唯一性了。

所以notCompare的作用就出来了。

```go
type DoNotCompare [0]func()

type T struct {
    name string
    age int
    DoNotCompare
}
func main() {
// ./cmp.go:13:21: invalid operation: T{} == T{} (struct containing DoNotCompare cannot be compared)
    fmt.Println(T{} == T{})
    fmt.Println(unsafe.Sizeof(T{}))	// 32
}
```

除了必须判断等于的结构体，我觉得项目中的大部分结构体都可以嵌套一个DoNotCompare，而且DoNotCompare不占用任何空间。



## NoUnkeyedLiterals

结构体初始化有两种：指定字段名称，或者按顺序列出所有字段，不指定名称

按顺序列出所有字段因为goland补全的原因，一直没这么用过，所以也没出现过新增字段不兼容的问题，但实际项目中估计还是要考虑有人有这么写的习惯问题，所以从根源解决问题可以一劳永逸。

```go
type User struct{
    Age int
    Address string
}

u := &User{21, "beijing"}
```

这么写当新增字段的时候会出现编译错误，这么写到时可以一开始就发现错误并且补上，但实际项目中，很少有人会改别人的代码，所以增加兼容性才是我们需要考虑的，也方便后面人继续接手。

```go
type User struct{
    Age int
    Address string
    Money int
}

func main(){
// ./struct.go:11:15: too few values in User{...}
  _ = &User{21, "beijing"}
}
```

这样就会报错。

如果我们增加一个空结构体嵌套在结构体内，切用占位符实现，就无法这么写了。

```go
type User struct{
    _ struct{}
    Age int
    Address string
}

func main(){
// ./struct.go:10:11: cannot use 21 (type int) as type struct {} in field value
// ./struct.go:10:15: cannot use "beijing" (type untyped string) as type int in field value
// ./struct.go:10:15: too few values in User{...}
_ = &User{21, "beijing"}
}
```

为了编译通过我们必须入一下这么写

```go
type User struct{
    _ struct{}
    Age int
    Address string
}

func main(){
	u := User{
		Age:     0,
		Address: "",
	}
}
```

