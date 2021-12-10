---
title: "cargo的使用"
date: 2021-12-08T17:19:15+08:00
draft: true
---



[官方文档](https://kaisery.github.io/trpl-zh-cn/ch01-03-hello-cargo.html)

### 1. 安装

在官方安装rustup的时候一般都会安装cargo

未安装查看 [安装rust](http://www.lsill.com/rust/%E5%AE%89%E8%A3%85rust/)



### 2. 使用

​	cargo 在我看来是一个规范化的工具，类似于go的一些cmd（gofmt）命令，不过go是放在语言里面，而rust是单独实现的。

- 创建工程 (cargo new hello_cargo)

- 

- ![cargo_create](E:\github\nbook\static\images\rust\cargo_create.PNG)

  ​	在hello_cargo中可以看到里面存放了一个Cargo.toml文件，一个src文件夹，一个.git(git仓库)，一个.gitignore（git忽视），直接创建了git工程是真的方便。

  Cargo.toml

  ```toml
  [package]
  name = "hello_cargo"
  version = "0.1.0"
  edition = "2021"
  
  # See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
  
  [dependencies]
  ```

  [package] 感觉像是一些介绍信息

  [dependencies] 像是go的package或者c++的头文件

  src是rust期望的一种代码存放规范，在我理解是源代码的仓库

  create的时候会自动创建src/main.rs(这块看起来是真的像go)

  ```rust
  fn main() {
      println!("Hello, world!");
  }
  ```

- 构建项目(cargo build)

  ​	这让我想起了go build,不过cargo build会在工程下创建一个target文件夹，target文件下存放了一些cargo需要的管理信息吧，debug文件夹下存放了可执行文件，这块的东西我没详细了解毕竟是个管理工具，有兴趣的可以看看cargo的[文档](https://doc.rust-lang.org/cargo/)

- 编译并运行可执行文件（cargo run）

  已经编译了直接运行，没有编译并运行

- 检查代码(cargo check)

  ​	检查代码写的对不对

- 创建release版本 （go build --release）

  这个创建release是真不错，发行版和开发分开，也省的在创建新的分支了

- 格式化代码(rustc fmt)

  ​	不格式化代码的话可能要解决一堆空格冲突问题

- 

