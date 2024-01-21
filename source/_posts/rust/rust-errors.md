---
title: Rust Learning-Errors
date: 2024-01-21 21:42:49
categories:
- programming
tags:
- rust
- learning
---

## RUST Error Handling 

[Rust 程序设计语言 - Rust 程序设计语言 简体中文版 (kaisery.github.io)](https://kaisery.github.io/trpl-zh-cn/title-page.html)

### 错误处理

rust中的错误分为不可恢复错误和可恢复错误两类。对于我们想知会用户的错误或重试操作的错误，是可恢复的，例如文件不存在。而越界访问一个数组是一个严重bug，就可以按不可恢复错误处理。

### Panic

rust中使用`panic!`宏来处理不可恢复的错误，出现panic后，程序会打印错误信息，清理函数栈然后退出程序。默认情况下，程序panic退出前，反向遍历每一个函数栈，释放其中的数据，然后再退出，这个过程需要一定时间，所以可以再release版本的程序中配置为直接退出程序，让操作系统来释放程序运行过程中申请的资源。

在项目的 *Cargo.toml*文件中增加以下代码，在release版本可以减少程序退出占用的时间：

```toml
[profile.release]
panic = 'abort'
```

可以直接调用`panic!`宏退出程序

```rust
panic!("I will be dead...");
// 程序会输出
//thread 'main' panicked at src\main.rs:17:5:
//I will be dead...
```

在C语言中访问数组越界时，程序还是会按实际的地址访问内存空间中的数据，只是错误是未定义的，这样会导致偶发不可预期的故障。rust中只要访问越界，就会panic，并明确告诉错误的原因。

```rust
let v = vec![1, 2, 3];
v[100]; // thread 'main' panicked at src\main.rs:18:6:
// index out of bounds: the len is 3 but the index is 100
```

通过在执行程序时设置`RUST_BACKTRACE=1` 环境变量，就可以把出错时的调用栈打印出来。

```shell
$ RUST_BACKTRACE=1 cargo run
或者
E:\dev\rust\cargo_demo\target\release>set RUST_BACKTRACE=1

E:\dev\rust\cargo_demo\target\release>cargo_demo.exe
thread 'main' panicked at src\main.rs:18:6:
index out of bounds: the len is 3 but the index is 100
stack backtrace:
   0: std::panicking::begin_panic_handler
             at /rustc/82e1608dfa6e0b5569232559e3d385fea5a93112/library\std\src\panicking.rs:645
   1: core::panicking::panic_fmt
   ......
```



### Result

enum Result有两种值，Ok用来表示成功返回值，Err表示失败时的返回值。

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

例如以下代码打开一个文件，它的返回值为  `Result<File, Error>`  ，然后可以使用match来处理两种情况，当Ok的时候，就把其中的file变量返回出去，如果失败了，就打印错误信息

```rust
use std::fs::File;
fn main() {
    let greeting_file_result = File::open("hello.txt");
    let greeting_file = match greeting_file_result {
        Ok(file) => file,  // Result是系统预置类型，所以不需要Resulat::前缀
        Err(error)=> panic!("Open file failed: {:?}", error),
        //Open file failed: Os { code: 2, kind: NotFound, message: "The system cannot find the file specified." }
    };
}
```



### Summary




