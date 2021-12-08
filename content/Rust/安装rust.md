---
title: "rust安装"
date: 2021-12-08T15:19:15+08:00
draft: true
---

### 1. window 安装

[下载](https://www.rust-lang.org/zh-CN/tools/install) 根据操作系统选择对应的rustup_init.exe

打开后 选择默认选项即可

![install_option](E:\github\nbook\static\images\rust\install_option.PNG)

安装成功

![r_ins_suc](E:\github\nbook\static\images\rust\r_ins_suc.PNG)

默认路径一般是C:\Users\用户名\\.cargo

环境变量一般会直接写入Path中，如果再命令行输入rustc --version失效，环境变量Path中加入C:\Users\用户名\\.cargo\bin 即可

输入rustc --version

![ru_ver_su](E:\github\nbook\static\images\rust\ru_ver_su.png)

找个喜欢的目录建立一个main.rs文件

```rust
fn main() {
    println!("Hello, world!");
}
```

执行下面命令输入就完全安装成功了

![r_hw_suc](E:\github\nbook\static\images\rust\r_hw_suc.PNG)

### 2. linux安装 // TODO 





### 3. mac安装 	// TODO

