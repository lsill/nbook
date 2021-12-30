---
title: "rust变量"
date: 2021-12-15T10:30:15+08:00
draft: true
---

[官方文档](https://kaisery.github.io/trpl-zh-cn/ch03-02-data-types.html)

### 标量类型

Rust有四种基本的标量类型：整形、浮点型、布尔类型和字符类型。

#### 整形

rust 中默认整数类型是i32

rust中的整数类型

| 长度    | 有符号 | 无符号 |
| ------- | ------ | ------ |
| 8-bit   | i8     | u8     |
| 16-bit  | i16    | u16    |
| 32-bit  | i32    | u32    |
| 64-bit  | i64    | u64    |
| 128-bit | i128   | u128   |
| arch    | isize  | usize  |

rust 中的整形字面值

| 数字字面值                 | 例子       |
| -------------------------- | ---------- |
| Decimal(十进制)            | 98_222     |
| Hex(十六进制)              | 0xff       |
| Octal(八进制)              | 0o77       |
| Binary(二进制)             | 0b111_0000 |
| Byte(单字节字符)(仅限于u8) | b'A'       |



#### 浮点型

rust 浮点书类型默认是f64 位

```rust
fn main() {
    let x = 2.0; // 没有显示声明 默认是 f64
    let y: f32 = 3.0; // 显示声明了 f32
}
```



#### 数值运算

```
fn main() {
    let sum = 5 + 10;  // 加
    println!("sum is {}", sum);
    let difference = 95.5 - 4.3;	// 减
    println!("difference is {}", difference);
    let product = 4 * 30;	// 乘
    println!("product is {}", product);
    let quotient = 56.7 / 32.2;	// 除（浮点数）
    println!("quotient is {}",quotient);
    let floored = 2 / 3;	// 除（整数）
    println!("floored is {}", floored);
    let remainder = 43 % 5;	// 取余
    println!("remainder is {}", remainder);
}
```

结果：

![data_type_cal](https://github.com/lsill/nbook/blob/main/static/images/rust/data_type_cal.PNG?raw=true)



#### 布尔型

两种声明方式 

```rust
fn main() {
    let t = true;
    println!("t is {}", t);
    let f: bool = false;
    println!("f is {}", f);
}
```



#### 字符类型

rust 的char 类型是语言中最原生的字母类型。

单引号声明字符类型，双引号声明字符串，rust 的char 类型的大小为四个字节（和golang的rune一样），大小4个字节代表能表示更多的内容。

```rust
fn main() {
    let _c = 'z';
    let _z = 'ℤ';
    let _heart_eyed_cat = '😻';
}
```



#### 复合类型

**复合类型**可以将多个值组合成一个类型，rust有两个原生的复合类型：元组（tuple）和数组(array)。



##### 元组类型

元组是一个将多个其他类型的值组合进一个复合类型的主要方式。元组长度固定：一旦声明，其长度不会增大或缩小。

我们使用包含在圆括号中的逗号分隔的值列表来创建一个元组。元组中的每一个位置都有一个类型，而且这些不同值的类型也不必是相同的。这个例子中使用了可选的类型注解：

```rust
fn main() {
    let tup: (i32, f64, u8) = (500, 6.4, 1);
}
```

`tup` 变量绑定到整个元组上，因为元组是一个单独的复合元素。为了从元组中获取单个值，可以使用模式匹配（pattern matching）来解构（destructure）元组值，像这样：

```rust
fn main() {
    let tup = (500, 6.4, 1, false);
    let (_x, y, _z, _u) = tup;
    println!("the value of y is {}", y);
}
```

程序首先创建了一个元组并绑定到 `tup` 变量上。接着使用了 `let` 和一个模式将 `tup` 分成了三个不同的变量，`x`、`y` 和 `z`。这叫做 **解构**（*destructuring*），因为它将一个元组拆成了三个部分。最后，程序打印出了 `y` 的值，也就是 `6.4`。

也可以使用点号（`.`）后跟值的索引来直接访问它们。例如：

```rust
fn main() {
    let tup = (500, 6.4, 1, false);
    let x = tup.0;
    let y = tup.1;
    let z = tup.2;
    let u = tup.3;
    println!("the x is {}, y is {}, z is {}, u is {}",x, y, z, u);
}
```

这个程序创建了一个元组，`tup`，并接着使用索引为每个元素创建新变量。

没有任何值的元组 `()` 是一种特殊的类型，只有一个值，也写成 `()` 。该类型被称为 **单元类型**（*unit type*），而该值被称为 **单元值**（*unit value*）。如果表达式不返回任何其他值，则会隐式返回单元值。



##### 数组类型

​	另一个包含多个值的方式是**数组**。与元组不同，数组中的每个元素的类型必须相同。Rust 中的数组与一些其他语言的数组不同，Rust中的数组长度是固定的。

​	我们将数组的值写成在方括号内，用逗号分隔：

```rust
fn main() {
    let a = [1,2,3,4,5];
}
```

当你想要在栈（stack）而不是在堆（heap）上为数据分配空间，或者是想要确保总是有固定数量的元素时，数组非常有用。但是数组并不如 vector(c++?) 类型灵活。vector 类型是标准库提供的一个 **允许** 增长和缩小长度的类似数组的集合类型。当不确定是应该使用数组还是 vector 的时候，那么很可能应该使用 vector.

确定元素个数不会改变时候，数组会更有用。

可以像这样编写数组的类型：在方括号中包含每个元素的类型，后跟分号，再后跟数组元素的数量。

```rust
let a: [i32; 5] = [1, 2, 3, 4, 5];
```

这里，`i32` 是每个元素的类型。分号之后，数字 `5` 表明该数组包含五个元素。

你还可以通过在放括号中指定初始值加分号再加元素个数的方式来创建一个每个元素都为相同值的数组：

```rust
let a = [3; 5];
```

变量名为 `a` 的数组将包含 `5` 个元素，这些元素的值最初都将被设置为 `3`。这种写法与 `let a = [3, 3, 3, 3, 3];` 效果相同，但更简洁。



数组访问：

```rust
use std::io;

fn main() {
    let a = [1, 2, 3, 4, 5];

    println!("Please enter an array index.");

    let mut index = String::new();

    io::stdin()
        .read_line(&mut index)
        .expect("Failed to read line");

    let index: usize = index
        .trim()
        .parse()
        .expect("Index entered was not a number");

    let element = a[index];

    println!(
        "The value of the element at index {} is: {}",
        index, element
    );
}
```

此代码编译成功。如果您使用 `cargo run` 运行此代码并输入 0、1、2、3 或 4，程序将在数组中的索引处打印出相应的值。如果你输入一个超过数组末端的数字，如 10，你会看到这样的输出：

```console
thread 'main' panicked at 'index out of bounds: the len is 5 but the index is 10', src/main.rs:19:19
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrac
```

程序在索引操作中使用一个无效的值时导致 **运行时** 错误。程序带着错误信息退出，并且没有执行最后的 `println!` 语句。当尝试用索引访问一个元素时，Rust 会检查指定的索引是否小于数组的长度。如果索引超出了数组长度，Rust 会 *panic*，这是 Rust 术语，它用于程序因为错误而退出的情况。这种检查必须在运行时进行，特别是在这种情况下，因为编译器不可能知道用户在以后运行代码时将输入什么值。

这是第一个在实战中遇到的 Rust 安全原则的例子。在很多底层语言中，并没有进行这类检查，这样当提供了一个不正确的索引时，就会访问无效的内存。通过立即退出而不是允许内存访问并继续执行，Rust 让你避开此类错误。第九章会讨论更多 Rust 的错误处理。



