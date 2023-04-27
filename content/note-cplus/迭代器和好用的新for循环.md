---
title: "迭代器和好用的新for循环"
date: 2023-02-15T19:17:53+18:00
draft: true
---

**现代C++实战30讲笔记**

### 什么是迭代器？
迭代器是一个很通用的概念，并不是一个特定的类型。它实际上是一组对类型的要求。它的最基本要求就是从一个端点出发，下一步、下一步地到达另一个端点。按照一般的中文习惯，也许“遍历”是比“迭代”更好的用词。我们可以遍历一个字符串的字符，遍历一个文件的内容，遍历目录里的所有文件，等等。这些都可以用迭代器来表达。
容器的 begin 和 end 成员函数返回的对象类型提出了要求。假设前者返回的类型是 I，后者返回的类型是 S，这些要求是：
- I对象支持*操作，解引用取的容器内的某个对象。
- I对象支持++，指向下一个对象。
- I对象可以和I或S对象进行相等比较，判断是否遍历到了特定位置（在S的情况下是 是否结束了遍历）。

注意在 C++17 之前，begin 和 end 返回的类型 I 和 S 必须是相同的。从 C++17 开始，I 和 S 可以是不同的类型。这带来了更大的灵活性和更多的优化可能性。

上面的类型I，多多少少就是一个满足**输入迭代器**（inpuit iterator）的类型了，输入迭代器要求前置和后置的++都得到支持。
输入迭代器不要求同一迭代器可以多次使用*运算符，也不要求可以保存迭代器来重新遍历对象。换句话，只要求可以单次访问。如果取消这些限制，允许多次访问的话，那迭代器同时满足了**前向迭代器**（forward iterator）。
一个前向迭代器的类型，如果同时支持--（前置及后置），回到前一个对象，那它就是个**双向迭代器**(bidirectional iterator)。也就是可以说，可以正常遍历，也可以反向遍历。
一个双向迭代器，如果额外支持在整数类型上的+、-、+=、-=，跳跃式地移动迭代器；支持[]，数组式的下标访问；支持迭代器的大小比较（之前只要求相等比较）；那么它就是个**随机访问迭代器。**（random-access iterator）
一个随机访问迭代器i和一个整数n，在*i可解引用且i+n是合法迭代的前提下，如果还额外满足*(addressdof(*i)+n)等价于*(i+n)，即保证迭代器指向的对象在内存里是连续存放的，那它（在C++20里）就是个**连续迭代器**。（contiguous iterator）
以上这些迭代器只考虑了读取。如果一个类型像输入迭代器，但 *i 只能作为左值来写而不能读，那它就是个**输出迭代器**（output iterator）。
而比输入迭代器和输出迭代器更底层的概念，就是迭代器了。基本要求是：
- 对象可以被拷贝构造、拷贝赋值和析构
- 对象自持*运算符
- 对象支持前置++运算符。

迭代器类型的关系可从下图中全部看到：
![itreator](https://raw.githubusercontent.com/lsill/nbook/dev/static/images/cplus/iterator.jpg)
迭代器通常是对象。但需要注意的是，指针可以满足上面所有的迭代器需求，因而也是迭代器。迭代器就是根据指针的特性，对其进行抽象的结果。事实上，vector的迭代器，在很多实现里就直接使用指针的。

### 常用迭代器
最常用的迭代器就是容器的iterator类型了。以顺序容器为例，它们定义了潜逃的iterator类型和const_iterator类型。一般而言，iterator可写入，const_iterator类型不可写入，但这些迭代器都被定义为输入迭代器或其派生类型：
- vector::iterator和array::iterator可以满足到连续迭代器。
- deque::iterator可以满足到随机访问迭代器（部分内存连续）。
- list::iterator可以满足双向迭代器。（链表不能快速跳转）
- forward_list::iterator可以满足到前向迭代器（单链表不能反向遍历）。

很常见一个输出迭代器是back_inserter返回的类型back_inserter_iterator了；用它可以很方便地在容器的尾部进行插入操作。另外一个常见的输出迭代器是ostream_iterator，方便我们把容器“拷贝”到一个输出流。示例如下：
```c++
#include <algorithm>
#include <iterator>
#include <vector>
#include <iostream>
#include "output_container.h"

using namespace std;

int main() {
  vector<int> v1{1,2,3,4,5};
  vector<int> v2;
  copy(v1.begin(),v1.end(), back_inserter(v2));
  cout<<v2 << endl; //{ 1, 2, 3, 4, 5 }
  copy(v2.begin(), v2.end(), ostream_iterator<int>(cout, " ")); //1 2 3 4 5 
}
```

### 使用时输入迭代器

先解说一下基于范围的 for 循环这个语法。虽然这可以说是个语法糖，但它对提高代码的可读性真的非常重要。如果不用这个语法糖的话，简洁性上的优势就小多了。
```c++
{
  auto&& r = istream_line_reader(is);
  auto it = r.begin();
  auto end = r.end();
  for (; it != end; ++it) {
    const string& line = *it;
    cout << line << endl;
  }
}
```
- 获取冒号后边的范围表达式的结果，并隐式产生一个引用，在整个循环期间都有效。注意根据生命周期延长规则，表达式结果如果是临时对象的话吗，这个对象要在循环结束后才被销毁。
- 自动生成遍历这个范围的迭代器。
- 循环内自动生成根据冒号左边的声明和*it来进行初始化语句。
- 下面就是完全正常的循环体。

生成迭代器这一步有可能是——但不一定是——调用r的begin和end成员函数，具体规则是：
- 对于C数组（必须是没有退化为指针的情况），编译器会自动生成指向数组头尾的指针（相当于自动应用可用于数组的std::begin和std::end函数）
- 对于有begin和end成员的对象，编译器会调用其begin和end成员函数（目前的情况）
- 否则，编译器会尝试在r对象所在的命名空间寻找可以用于r的begin和end函数，并调用begin(r)和end(r)；找不到的话则失败报错。

### 定义输入迭代器
要实现这个输入迭代器，需要做什么工作。
C++里有些固定的类型要求规范。对于一个迭代器，需要定义下面的类型：
```c++
class istream_line_reader {
 public:
  class iterator {
    typedef ptrdiff_t difference_type;  // 是代表迭代器之间距离的类型
    typedef string value_type;  // 迭代器指向的是字符串
    typedef const value_type* pointer;  // 是迭代器指向的对象的指针类型
    typedef const value_type& reference;  // reference 是 value_type 的常引用
    typedef input_iterator_tag iterator_category; // 标识这个迭代器的类型是input_iterator
  };
};
```
仿照一般的容器，把迭代器定义为istream_line_reader的嵌套类。它里面的这五个类型是必须定义的（其他泛型C++代码可能会用到这五个类型；之前标准库定义了一个可以继承的类模版std::iterator来产生这些类型定义，但这个类已经被废弃）。其中：
- difference_type是代表迭代器之间距离的类型，定义为ptrdiff_t只是种标准做法（指针间差值的类型），对这个类型没什么特别作用。
- value_type是迭代器指向的对象的值类型，我们使用string，表示迭代器指向的是字符串。
- pointer是迭代器指向的对象的指针类型，这儿就平淡无奇的定义为value_type的常指针了（不希望别人来更改指针的内容）。
- 类似的，reference是value_type的常引用。
- iterator_category被定义为input_iterator_tag，标识这个迭代器的类型是input_iterator（输入迭代器）

作为一个真的只能读一次的输入迭代器，有个特殊的麻烦（前向迭代器或其衍生类型没有）：到底应该让 * 负责读取还是 ++ 负责读取。我们这儿采用常见、也较为简单的做法，让 ++ 负责读取，* 负责返回读取的内容（这个做法会有些副作用，但按我们目前的用法则没有问题）。这样的话，这个 iterator 类需要有一个数据成员指向输入流，一个数据成员来存放读取的结果。根据这个思路，我们定义这个类的基本成员函数和数据成员：
```c++
class istream_line_reader {
public:
  class iterator {
    …
    iterator() noexcept
      : stream_(nullptr) {} // 默认构造函数，将 stream_ 清空
    explicit iterator(istream& is)  // 根据传入的输入流来这是stream_
      : stream_(&is)
    {
      ++*this;
    }
    reference operator*() const noexcept
    {
      return line_;
    }
    pointer operator->() const noexcept
    {
      return &line_;
    }
    iterator& operator++()  // ++来读取输入流的内容
    {
      getline(*stream_, line_);
      if (!*stream_) {
        stream_ = nullptr;
      }
      return *this;
    }
    iterator operator++(int)    // 后置++则以惯常方式使用前置++和拷贝构造来实现
    {
      iterator temp(*this);
      ++*this;
      return temp;
    }
  private:
    istream* stream_;
    string line_;
  };
  …
};
```
定义了默认的构造函数，将steam_清空；相应的，在带参数的构造函数里，根据传入的输入流来这是stream_。定义了*和->运算符来取的迭代器指向的文本的引用和指针，并用++来读取输入流的内容（后置++则以惯常方式使用前置++和拷贝构造来实现）。唯一“特别”的地方，是在构造函数里调用了++，确保在构造函数调用*运算符时可以读取内容，符合日常先使用*再使用++的习惯。一旦文件读取到尾部（或出错），则stream_被清空，回到默认构造的情况。
对于迭代器之间的比较，我们则主要考虑文件有没有读到尾部的情况，简单定义为：
```c++
  bool operator==(const iterator& rhs)
      const noexcept
    {
      return stream_ == rhs.stream_;
    }
    bool operator!=(const iterator& rhs)
      const noexcept
    {
      return !operator==(rhs);
    }
```
有了这个 iterator 的定义后，istream_line_reader 的定义就简单得很了：
```c++
class istream_line_reader {
public:
  class iterator {…};
  istream_line_reader() noexcept
    : stream_(nullptr) {}
  explicit istream_line_reader(
    istream& is) noexcept
    : stream_(&is) {}
  iterator begin()
  {
    return iterator(*stream_);
  }   iterator end() const noexcept
  {
    return iterator();
  }
private:
  istream* stream_;
};
```
也就是说，构造函数只是简单的把输入流的指针赋值给stream_成员变量。begin成员函数则负责构造一个真正有意义的迭代器；end成员函数则只是返回一个默认构造函数的迭代器。

### 思考
1. 目前这个输入行迭代器的行为，在什么情况下可能导致意料之外的后果？

2. 请尝试一下改进这个输入行迭代器，看看能不能消除这种意外。如果可以，该怎么做？如果不可以，为什么？