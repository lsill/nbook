---
title: "SFINAE：不是错误的替换失败是怎么回事?"
date: 2023-03-02T17:16:45+18:00
draft: true
---

**现代C++实战30讲笔记**

### 函数模版的重载决议
当一个函数名称和某个函数模版名称匹配时，重载决议过程大致如下：
- 根据名称找出所有适用的函数和函数模版
- 对于使用的函数模版，要根据实际情况对模版形参进行替换；替换过程中如果发生错误，这个模版会被丢弃
- 在上面两部生成的克星函数集合中，编译器会寻找一个最佳匹配，产生对该函数的调用
- 如果没有找到最佳匹配，或者找到多个匹配程度相当的函数，则编译器需要报错

来看一个具体的例子（改编自参考资料 ）。虽然这例子不那么实用，但还是比较简单，能够初步说明一下。
```c++
#include <stdio.h>
struct Test {
  typedef int foo;
};
template <typename T>
void f(typename T::foo)
{
  puts("1");
}
template <typename T>
void f(T)
{
  puts("2");
}
int main()
{
  f<Test>(10);
  f<int>(10);
}
```
输出为：
```
1
2
```
f<Test>(10)分析：
- 有两个模版符合名字 f
- 替换结果为f(Test::foo)和f(Test)
- 使用参数10去匹配，只有前者参数可以匹配，因而第一个模版被选择

在看f<int>(10)的情况：
- 有两个模版符合名字 f
- 替换结果为f(int::foo)和f(int)；显然前者不是个合法的类型，被抛弃
- 使用参数10去匹配f(int)，没有问题，那就使用这个模版示例了

在这儿，体现的是 SFINAE 设计的最初用法：如果模板实例化中发生了失败，没有理由编译就此出错终止，因为还是可能有其他可用的函数重载的。

这儿的失败仅指函数模板的原型声明，即参数和返回值。函数体内的失败不考虑在内。如果重载决议选择了某个函数模板，而函数体在实例化的过程中出错，那我们仍然会得到一个编译错误。

### 编译期成员检测
不过，很快人们就发现 SFINAE 可以用于其他用途。比如，根据某个实例化的成功或失败来在编译期检测类的特性。下面这个模板，就可以检测一个类是否有一个名叫 reserve、参数类型为 size_t 的成员函数：
```c++
template <typename T>
struct has_reserve {
  struct good { char dummy; };
  struct bad { char dummy[2]; };
  template <class U,
            void (U::*)(size_t)>  // 模版的第二个参数需要是第一个参数的成员函数指针，并且参数类型是size_t，返回值是void
  struct SFINAE {};
  template <class U>
  static good reserve(SFINAE<U, &U::reserve>*);
  template <class U>
  static bad reserve(...);
  static const bool value =
    sizeof(reserve<T>(nullptr))
    == sizeof(good);
};
```
在这个模版里：
- 首先定义了两个结构good和bad；它们的内容不重要，只关心它们的大小必须不一样。
- 然后定义了一个SFINAE模版，内容同样也不重要，但模版的第二个参数需要是第一个参数的成员函数指针，并且参数类型是size_t，返回值是void
- 随后，定义了一个要求SFINAE*类型的reserve成员函数模版，返回值是good；在定义了一个对参数类型无要求的reserve成员函数模版，返回值是bad
- 最后，定义常整形布尔值value，结果是true还是false，取决于nullptr能不能和SFINAE*匹配成功，而这又取决于模版参数T有没有返回类型void、接受一个参数并且类型为size_t的成员函数reserve.

那这样的模板有什么用处呢？

### SFINAE模版技巧

1. enable_if

C++11开始，标准库有了一个叫enable_if的模版（定义在<type_traits>里），可以用它来选择性的启用某个函数的重载。

假设有一个函数，用来往一个容器尾部追加元素，希望原型是这个样子的：
```c++
template<typename C, typename T>
void append(C& container, T* ptr, size_t size);
```
显然，container有没有reserve成员函数，是对性能有影响的——如果有的话，通常应该预留好内存空间，以免产生必须要的对象移动或者拷贝操作，利用enable_if和上面的has_reserve模版，可以这么写：
```c++
template <typename C, typename T>
std::enable_if_t<has_reserve<C>::value,
            void>
append(C& container, T* ptr,
       size_t size){
  container.reserve(
      container.size() + size);
  for (size_t i = 0; i < size;
       ++i) {
    container.push_back(ptr[i]);
  }
}
template <typename C, typename T>
std::enable_if_t<!has_reserve<C>::value,
            void>
append(C& container, T* ptr,
       size_t size){
  for (size_t i = 0; i < size;
       ++i) {
    container.push_back(ptr[i]);
  }
}
```
对于某个type trait，添加_t的后缀等驾驭其type成员类型。因而，可以用enable_if_v来取到结果的类型。

enable_if_t<has_reserve<C>::value, void>的意思可以理解成：如果类型C有reserve成员的话，那启用下面的成员函数，它的返回类型为void。

2. decltype返回值

如果只需要在某个操作有效的情况下启用某个函数，而不需要考虑相反的情况的话，有另外一个技巧可以用。对于上面的append的情况，如果想限制只有具有reserve成员函数的类可以使用这个重载，可以把代码简化成：
```c++
template <typename C, typename T>
auto append(C& container, T* ptr,size_t size) -> decltype(declval<C&>().reserve(1U),void()){
  container.reserve(container.size() + size);
  for (size_t i = 0; i < size;++i) {
    container.push_back(ptr[i]);
  }
}
```
`declval`这个模版用来声明某一个类型的参数，但这个参数只是用来参加模版的匹配，不允许实际使用。使用这个模版，可以在某类型没有默认构造函数的情况下，假想出一个该类的对象来进行类型推导。declval<C&>().reserve(1U)用来测试C&类型的对象是不是可以拿1U作为参数来调用reserve成员函数。C++里的逗号表达式的意思是按顺序逐个估值，并返回最后一项。所以上面这个函数返回值类型是void。

这个方式和enable_if不同，很难表示否定的条件。如果要提供一个专门给没有reserve成员函数的C类型append重载，这种方式就不太方便了。因而，这种方式的主要用途是避免错误的重载。

3. void_t
`void_t`是C++17新引入的一个模版，它的定义简单的令人吃惊：
```c++
template<typename ...>
using void_t = void;
```
换句话说，这个类型模版会把任意类型映射到void。它的特殊性在于，在这个看似无聊的过程中，编译器会检查那个“任意类型”的有效性。利用decltype、declval和模版特化，可以把has_reserve的定义大大简化：
```c++
template<typename T, typename = std::void_t<>>
struct has_reserve : false_type {};

template<typename T>
struct has_reserve<T, std::void_t<decltype(std::declval<T&>().reserve(1U))>> : true_type {};
```
这里第二个has_reserve模版的定义实际是一个偏特化。偏特化师类模版的特有功能，跟函数重载有些相似。编译器会找出所有的可用模版，然后选择其中最“特别”的一个。像上面的例子，所有类型都能满足第一个模版，但不是所有的类型都能满足第二个模版，所有第二个更特别。当第二个模版能被满足时，编译器就会选择第二个特化的模版；而只有第二个模版不能被满足时，才会回到第一个模版的通用情况。

有了这个has_reserve模版，就可以继续使用其他的技巧，如enable_if和下面标签分发，来对重载进行限制。

4. 标签分发
在上一讲，我们提到了用 true_type 和 false_type 来选择合适的重载。这种技巧有个专门的名字，叫标签分发（tag dispatch）。我们的 append 也可以用标签分发来实现：
```c++
template<typename C, typename T>
void _append(C& container, T* ptr, size_t size,true_type) {
  container.reserve(container.size()+size);
  for (size_t i = 0; i < size;++i) {
    container.push_back(ptr[i]);
  }
}

template<typename C, typename T>
void _append(C& container, T* ptr, size_t size, false_type) {
  for (size_t i = 0; i < size;++i) {
    container.push_back(ptr[i]);
  }
}

template<typename C, typename T>
void _append(C& container, T* ptr, size_t size) {
  _append(container,ptr,size,integral_constant<bool, has_reserve<C>::value>{});
}
```
这个代码定义喝enable_if是等价的。

如果使用void_t那个版本的has_reserve模版的话，由于模版的实例会继承false_type或true_type之一，代码可以进一步简化为：
```c++
template<typename C, typename T>
void _append(C& container, T* ptr, size_t size) {
  _append(container, ptr,size, has_reserve<C>{});
}
```

### 静态多态的限制？
看到这儿，你可能会怀疑，为什么我们不能像在 Python 之类的语言里一样，直接写下面这样的代码呢？
```c++
template<typename C, typename T>
void append(C& container, T* ptr, size_t size) {
  if (has_reserve<C>::value) {
    container.reserve(container.reserve() + size);
  }
  for (size_t i = 0; i < size;++i) {
      container.push_back(ptr[i]);
  }
}
```

如果试验一下，就会发现，在 C 类型没有 reserve 成员函数的情况下，编译是不能通过的，会报错。这是因为 C++ 是静态类型的语言，所有的函数、名字必须在编译时被成功解析、确定。在动态类型的语言里，只要语法没问题，缺成员函数要执行到那一行上才会被发现。这赋予了动态类型语言相当大的灵活性；只不过，不能在编译时检查错误，同样也是很多人对动态类型语言的抱怨所在……

