---
title: "string和[]byte的零拷贝"
date: 2022-01-05T17:27:38+08:00
draft: false
---



很多时候我们都是如下直接转换字符串和字节数组的，但是下面这种强转方式无疑需要重新分配内存的，有分配内存就有消耗、

```golang
str := string(array)	// 字节数组转字符串
array := []byte(str)	// 字符串转字节数组
```



在`golang`中，切片和字符串之间的数据结构相似但不相同，中间差了一个cap容器大小。

切片结构

```golang
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

字符串结构

```go
type stringStruct struct {
	str unsafe.Pointer
	len int
}
```

都包含一个`unsafe`指针和int类型的长度。

一个`string`和一个`[]byte`都有一个指向字节切片的指针，我们是否可以直接强转类型来节省空间分配的消耗？

```go
func StringToBytes(str string) []byte {
	return *(*[]byte)(unsafe.Pointer(&s))
}

func BytesToString(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}
```

如上的转换方式是通过先取到指向对应结构的指针然后再强转指针类型取出里面的内容来完成一次零拷贝。但上述的方式存在一定问题，如果是字节切片转换成字符串没有问题，结构减少`cap`没有什么影响，但字符串强转字符切片就存在问题了，切片中的`cap`的值会不确定，会有未知错误。如果是字符切片转字符串啥问题。



unsafe包提供了一种比较安全的强转方式

```go
func SliceToString(b []byte) (s string) {
	pBytes := (*reflect.SliceHeader)(unsafe.Pointer(&b))
	pString := (*reflect.StringHeader)(unsafe.Pointer(&s))
	pString.Data = pBytes.Data
	pString.Len = pBytes.Len
	return
}

func StringToSlice(s string) (b []byte) {
	pBytes := (*reflect.SliceHeader)(unsafe.Pointer(&b))
	pString := (*reflect.StringHeader)(unsafe.Pointer(&s))
	pBytes.Data = pString.Data
	pBytes.Len = pString.Len
	pBytes.Cap = pString.Len
	return
}
```



当然`go` 中始终不提倡用`unsafe`包，如果不是性能瓶颈的话，还是不建议使用。
