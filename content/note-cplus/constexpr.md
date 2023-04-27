---
title: "constexpr：一个常态的世界"
date: 2023-03-03T17:26:32+18:00
draft: true
---

### 初识 constexpr
先看一些例子：
```c++
int sqr(int n) {
  return n*n;
}

int main() {
  int a[sqr(3)];
}
```
这个代码合法吗？ （20能通过编译）
在看这个代码
```c++
int sqr(int n) {
  return n*n;
}

int main() {
  const int n = sqr(3);
int a[sqr(3)];
}
```
（20能通过编译）
还有这个：
```c++
int sqr(int n) {
  return n*n;
}

int main() {
  std::array<int, sqr(3)> a;
}
```
(编译无法通过)
此外，我们前面模板元编程里的那些类里的 static const int 什么的，你认为它们能用在上面的几种情况下吗？

如果以上问题你都知道正确的答案，那恭喜你，你对 C++ 的理解已经到了一个不错的层次了。但问题依然在那里：这些问题的答案不直观。并且，我们需要一个比模板元编程更方便的进行编译期计算的方法。

在 C++11 引入、在 C++14 得到大幅改进的 constexpr 关键字就是为了解决这些问题而诞生的。它的字面意思是 constant expression，常量表达式。存在两类 constexpr 对象：

- constexpr 变量
- constexpr 函数

一个 constexpr 变量是一个编译时完全确定的常数。一个 constexpr 函数至少对于某一组实参可以在编译期间产生一个编译期常数。

注意一个 constexpr 函数不保证在所有情况下都会产生一个编译期常数（因而也是可以作为普通函数来使用的）。编译器也没法通用地检查这点。编译器唯一强制的是：
- constexpr 变量必须立即初始化
- 初始化只能使用字面量或常量表达式，后者不允许调用任何非constexpr函数

constexpr 的实际规则当然稍微更复杂些，而且随着 C++ 标准的演进也有着一些变化，特别是对 constexpr 函数如何实现的要求在慢慢放宽。要了解具体情况包括其在不同 C++ 标准中的限制，可以查看参考资料 。下面我们也会回到这个问题略作展开。

拿 constexpr 来改造开头的例子，下面的代码就完全可以工作了：
```c++
constexpr int sqr(int n) {
  return n*n;
}

int main() {
  constexpr int n = sqr(3);
  std::array<int, n> a;
  int b[n];
}
```

要检验一个 constexpr 函数能不能产生一个真正的编译期常量，可以把结果赋给一个 constexpr 变量。成功的话，我们就确认了，至少在这种调用情况下，我们能真正得到一个编译期常量。

### constexpr和编译器计算
上面这些当然有点用。但如果只有这点用的话，就不值得专门来写一讲了。更强大的地方在于，使用编译期常量，就跟之前的那些类模板里的 static const int 变量一样，是可以进行编译期计算的。

等价的阶乘函数：
```c++
constexpr int factorial_1(int n) {
  if (n == 0) {
    return 1;
  } else {
    return n * factorial_1(n-1);
  }
}
```
然后，用下面的代码可以验证我们确实得到了一个编译期常量：
```c++
int main()
{
  constexpr int n = factorial_1(10);
  printf("%d\n", n);
}
```
编译可以通过，同时，如果看产生的汇编代码，一样可以直接看到常量3628800。

这里有一个问题：在这个constexpr函数里，是不能写static_assert(n>=0)的。一个constexpr函数仍然可以作为皮桶函数使用————显然，传入一个普通int是不能使用静态断言的。替换方法是在factorial_1的实现开头加入：
```c++
constexpr int factorial_1(int n) {
  if (n < 0) {
    throw std::invalid_argument("Arg must be non-negative");
  }
  if (n == 0) {
    return 1;
  } else {
    return n * factorial_1(n-1);
  }
}
```
如果在 main 里写 constexpr int n = factorial(-1); 的话，就会看到编译器报告抛出异常导致无法得到一个常量表达式。

### constexpr和const
初学 constexpr 时，一个很可能有的困惑是，它跟 const 用法上的区别到底是什么。产生这种困惑是正常的，毕竟 const 是个重载了很多不同含义的关键字。

const的原本和基础的含义，自然是表示它修饰的内容不会变化，如：
```c++
const int n = 1;
n = 2;// 出错
```
注意const在类型声明的不同位置会产生不同的结果。对于常见的const char*这样的类型声明，意义和char const*相同，是指向常字符的指针，指针指向的内容不可更改；但和char* const不同那代表指向字符的常指针，指针本身不可更改。本质上，const用来表示**运行时常量**。

在 C++ 里，const 后面渐渐带上了现在的 constexpr 用法，也代表编译期常数。现在——在有了 constexpr 之后——我们应该使用 constexpr 在这些用法中替换 const 了。从编译器的角度，为了向后兼容性，const 和 constexpr 在很多情况下还是等价的。但有时候，它们也有些细微的区别，其中之一为是否内联的问题。

### 内联变量
C++17引入了内联壁变量的概念，允许在头文件中定义内联变量，然后像内联函数一样，只要所有的定义都相同，那变量的定义出现多次也没有关系。对于类的静态数据成员，const缺省是不内联的，而constexpr缺省缺省就是内联的。区别在用&去取一个const int值的地址、或将其传到一个形参类型为const int&的函数去的时候（这在C++文档里的行话叫ODR-use），就会体现出来。
下面是个合法的完整程序：
```c++
struct magic {
    static const int number = 42;
};

int main() {
  std::cout << magic::number << std::endl;
}
```
稍微改一点
```c++
struct magic {
    static const int number = 42;
};

int main() {
  std::vector<int> v;
  v.push_back(magic::number);
  std::cout << v[0] << std::endl;
}
```
程序在链接时就会报错了，说找不到magic::number（注意：MCSVC缺省不报错，但使用标准模式——/Za命令行选项——也会出现这个问题）。这是因为ODR-use的类静态常量也需要有一个定义，在没有内联变量之需要在某一个源文件（非头文件）中这样写：
```c++
const int magic::number = 42;
```
必须正正好好一个，多了少了都不行，所以叫 one definition rule。内联函数，现在又有了内联变量，以及模板，则不受这条规则限制。

修正这个问题的简单方法是把magic里的static const 改成static constexpr或static inline const。前者可行的原因是，**类的静态constexpr成员变量默认就是内联的。const常量和类外边的constexpr变量不默认内联，需要手工加inline关键字才会变成内联。**

### constexpr变量模版
变量模版是C++14引入的新概念。之前需要用类静态数据成员来表达的东西。使用变量模版可以更简洁地表达。constexpr很适合用在变量模版里，表达一个和某个类型相关的编译期常量。由此，type traits都获得了一种更简单的表达方式。
```c++
template<class T>
inline constexpr bool is_trivially_destructible_v = std::is_trivially_destructible<T>::value;
```
了解变量也可以是模版后，上面这个代码就容易看懂了。这只是一个小小的语法糖，允许把 std::is_trivially_destructible<T>::value写成is_trivially_destructible_v<T>


### constexpr变量仍是const
一个constexpr变量仍然是const常类型。需要注意的是，就像const char*类型是指向常量的指针、自身不是const常量一样，下面这个表达式里的const也是不能缺少的：
```c++
constexpr int a = 42;
constexpr const int& b = a;
```
第二行里，constexpr表示b是一个编译器常量，const表示这个引用是常量引用。去掉这个const的话，编译器就会认为是试图将一个普通引用绑定到一个常数上，报一个类似于下面的错误信息：
    
    error: binding reference of type ‘int&’ to ‘const int’ discards qualifiers

如果按照 const 位置的规则，constexpr const int& b 实际该写成 const int& constexpr b。不过，constexpr 不需要像 const 一样有复杂的组合，因此永远是写在类型前面的。


### constexpr构造函数和字面类型
一个合理的constexpr函数，应当至少对于某一组编译器常量的输入，能得到编译器常量的结果，为此，对这个函数也是有些限制的：
- 最早，constexpr函数里循环都不有，但C++14放开了
- 目前，constexpr函数仍不能有try...catch语句和asm声明，但C++20会放开。
- constexpr函数里不能使用goto语句。
- 等等。

一个有意思的情况是一个类的构造函数。如果一个类的构造函数里面只包含常量表达式、满足对constexpr函数的限制的话（这也意味着，里面不可以有任何动态内存分配），并且类的析构函数是平凡的，那么整个类就可以被成为是一个字面类型。换一个角度想，对constexpr函数——包括字面类型构造函数——要求是，得让编译器能在编译器进行计算，而不会产生任何“副作用”，比如内存分配、输入、输出等等。

为了全面支持编译期计算，C++14开始，很多标准类的构造函数和成员函数已经标为constxepr，一边在编译期使用。当然，大部分的容器类，因为用到了动态内存分配，不能成为字面类型。下面这些不实用动态内存分配的字面类型则可以在常量表达式中使用：
- array
- initializer_list
- pair
- tuple
- string_view
- optional
- variant
- bitset
- complex
- chrono::duration
- chrono::time_point
- shared_ptr(仅默认构造和空指针构造)
- unique_ptr(仅默认构造和空指针构造)
- ...

下面这个玩具例子，可以展示上面的若干类及其成员函数的行为：
```c++
int main() {
  constexpr std::string_view sv{"hi"};
  constexpr std::pair pr {sv[0],sv[1]};
  constexpr std::array a {pr.first, pr.second};
  constexpr int n1 = a[0];
  constexpr int n2 = a[1];
  std::cout << n1 << ' ' << n2 << '\n';
}
```
编译器可以在编译器决定n1和n2的数值；从最后结果的角度，上面程序就是输出了两个整数而已。

### if constexpr
在类型参数 C 没有 reserve 成员函数时不能编译的代码：
```c++
template<typename C, typename T>
void append(C& container, T* ptr, size_t size) {
  if (has_reserve<C>::value) {
    container.reserve(container.size()+size);
  }
  for (size_t i = 0;i<size;++i) {
    container.push_back(ptr[i]);
  }
}
```
在C++17里，只要在if后面加上constexpr，代码就可以工作了。当然，它要求括号里的条件是个编译期常量。满足这个条件后，标签分发、enable_if那些技巧就不那么用了。显然，使用If constexpr能比使用那些其他方式，写出更可读的代码...

### output_container.h解读
解读output_container.h用到的语法特性。
```c++
#include <type_traits>

template<typename T>
struct is_pair:std::false_type{};
template <typename T, typename U>
struct is_pair<std::pair<T, U>>:std::true_type {};

template<typename T>
inline constexpr bool is_pair_v = is_pair<T>::value;
```
这段代码利用模版特化和false_type、true_type类型，定义了is_pair，用来检测一个类型是不是pair。随后，定义了内联constexpr变量is_pair_v，用来简化表达。
```c++
template<typename T>
struct has_output_function {
  template <class U>
  static auto output(U *ptr) -> decltype(std::declval<std::ostream &>() << *ptr, std::true_type{});

  template <class U>
  static std::false_type output(...);

  static constexpr bool value = decltype(output<T>(nullptr))::value;
};

template<typename T>
inline constexpr bool has_output_function_v = has_output_function<T>::value;
```
这段代码使用 SFINAE技巧，来检测模版参数T的对象是否可以直接输出到ostream。然后，一样用一个内联constexpr变量来简化表达。
```c++
// Output function for std::pair
template<typename T, typename U>
std::ostream& operator<<(std::ostream& os, const std::pair<T,U>& pr);
```
然后在声明了一个pair的输出函数（标准库没有提供这个功能）。这儿只是声明，是因为这儿有两个输出函数，且可能互相调用。所以要声明其中之一。

下面会看到，pair的通用输出形式是"(x,y)"。
```c++

```




