---
title: "rust变量"
date: 2021-12-14T16:55:15+08:00
draft: false
---



[官方文档](https://kaisery.github.io/trpl-zh-cn/ch03-01-variables-and-mutability.html)

### 1. 变量

​	变量默认是不可改变的，但仍然可以试用可变变量（rust不鼓励使用）。

​	这个和很多语言不一样了，变量的声明如果没有加mut关键词,那申请的变量相当于一个常量了。

```rust
fn main() {
    let x = 5;
    println!("The value of x is: {}", x);
    x = 6;	// 这一行编译器会提示有错误，因为是变量，所以不可。。变。。难搞
    println!("The value of x is: {}", x);
}
```

cargo run显示的报错

![rust_var_1](E:\github\nbook\static\images\rust\rust_var_1.PNG)

不能对不可变的变量二次赋值。。（好不习惯啊）

用rust说法：Rust 编译器保证，如果声明一个值不会变，它就真的不会变，所以你不必自己跟踪它。这意味着你的代码更易于推导。



如果要使用可变的变量的话，那么可以用mut关键词来修饰

```rust
fn main() {
    let mut x = 5;  // x可以被改变了
    println!("The value of x is: {}", x);
    x = 6;
    println!("The value of x is: {}", x);
}

```

cargo run 运行结果

![rust_var_1_succ](E:\github\nbook\static\images\rust\rust_var_1_succ.PNG)

**原文对于变量什么时候使用mut什么时候不使用是这样描述的：**

​	通过 `mut`，允许把绑定到 `x` 的值从 `5` 改成 `6`。除了防止出现 bug 外，还有很多地方需要权衡取舍。例如，使用大型数据结构时，适当地使用可变变量，可能比复制和返回新分配的实例更快。对于较小的数据结构，总是创建新实例，采用更偏向函数式的编程风格，可能会使代码更易理解，为可读性而牺牲性能或许是值得的。

**自己理解的**

​	对于数据量大的结构体，比如一个user（里面有账号id，名字，各种属性字段，各种自定义结构体），我们只会再登录的时候去实例化这个user，而且user的数据变化非常频繁，需要可以考虑变更的时候直接修改内存值就好了。而对于一些比较简单的结构，比如一些部分表数据或者配置数据，一般情况下不会出现更改，如果出现更改，在rust眼里重新申请一分新的实例，要比修改原实例的值更容易规避bug。



### 2.常量

​	常量在所有语言都是不可变量的意思，不可被修改。

​	在rust内也是用const关键词来定义的

​	常量可以再任何作用域声明。（错误码一般都用得常量，比如404）

​	rust对常量的命名约束是全大写加下划线。

声明一个常量

```rust
const THREE_HOURS_IN_SECONDS:u32 = 60 * 60 * 3; // 任何作用域来定义
fn main() {
    let mut x = 5;  // x可以被改变了
    println!("The value of x is: {}", x);
    x = 6;
    println!("The value of x is: {}", x);
}

```

上面的代码常量THREE_HOURS_IN_SECONDS定义没有被使用，讲道理是不该被编译过得，但可以被编译过，不过会打印出警告的日志，警告常量THREE_HOURS_IN_SECONDS是一个dead_code

![rust_var_const](E:\github\nbook\static\images\rust\rust_var_const.PNG)

### 3.变量的隐藏

```rust
fn main() {
    let x = 5;  // x1
    let x = x +1 ;  // x2,x1被x2隐藏
    {
        let x = x * 2;  // x3,在{}作用域内x2被x3隐藏
        println!("the value of x in the inner scope is: {}",x);
    }
    println!("the value of x is :{}",x);    // 此处是x2
}
```

输出值：

![rust_var_shadowing1](E:\github\nbook\static\images\rust\rust_var_shadowing1.PNG)

看起来就像一些语言的作用域，同名变量在不同作用域下上层作用域的值会被隐藏

重点：**隐藏甚至可以改变变量的类型，这和其他语言的作用域不同**

可以通过编译

```rust
let spaces = "   ";
let spaces = spaces.len(); 	// 隐藏
```

不可通过编译

```
let mut spaces = "   ";
spaces = spaces.len(); // 变量类型不可以被修改
```

