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
use std::io::ErrorKind;

fn main() {
    let greeting_file_result = File::open("hello.txt");
    let greeting_file = match greeting_file_result {
        Ok(file) => file, // Result是系统预置类型，所以不需要Result::前缀
        Err(error) => match error.kind() {
            ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(fc) => fc, // 打不开文件后，创建文件
                Err(e) => panic!("Problem creating the file: {:?}", e),
            },
            other_error => {
              	// 让程序退出
                panic!("Problem opening the file: {:?}", other_error);
            }
        },
    };
}
```

#### unwrap

`Result<T, E>` 的unwrap()方法简化错误处理，它内部实现了Ok时返回结果，错误时调用默认的panic。

```rust
use std::fs::File;

fn main() {
    // called `Result::unwrap()` on an `Err` value: 
    // Os { code: 2, kind: NotFound, message: "The system cannot find the file specified." }
    let greeting_file = File::open("hello.txt").unwrap();
}
```

#### expect

`Result<T, E>`的expect()方法它内部实现了Ok时返回结果，错误时可以指定的panic的信息

```rust
use std::fs::File;

fn main() {
    let greeting_file = File::open("hello.txt")
        .expect("hello.txt should be included in this project");
    //thread 'main' panicked at src\main.rs:5:10:
    //hello.txt should be included in this project: 
    //Os { code: 2, kind: NotFound, message: "The system cannot find the file specified." }       
}
```

#### 传播错误

把函数执行过程中的错误返回给调用者，这样调用者可以看情况处理错误。可以使用match来直接返回错误

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error>{
    let username_file_result = File::open("hello.txt");
    let mut username_file = match username_file_result {
        Ok(file) => file,
        Err(e) => return Err(e), // 直接返回错误
    };

    let mut username = String::new();
    match username_file.read_to_string(&mut username) {
        Ok(_) => Ok(username),
        Err(e) => Err(e), // 返回错误状态
    }
}
```

为了简化match语句返回错误，rust支持使用`?`操作符来提前返回错误。在一个函数调用结束时使用？，如果函数返回错误，就直接返回错误，不用写match。

```rust
fn read_username_from_file() -> Result<String, io::Error> {
    let mut username_file = File::open("hello.txt")?;// 直接返回错误或正常返回文件句柄
    let mut username = String::new();
    username_file.read_to_string(&mut username)?;
    Ok(username)
}
```

 `?` 会调用实现了`From`Trait的结构体的`from`方法，把调用函数返回的错误类型转换为当前？所在函数返回的错误类型。

使用 `?` 后，可以方便的写链式调用。

```rust
fn read_username_from_file() -> Result<String, io::Error> {
    let mut username = String::new();
    File::open("hello.txt")?.read_to_string(&mut username)?;
    Ok(username)
}
```

 `?` 操作符只能用于返回 `Result` 或 `Option` (或其他类型实现了 `FromResidual`)的函数，例如上面的函数返回值为Result.例如下面的会有编译错误，因为main的返回值类型为`()`

```rust
fn main() {
    let greeting_file = File::open("hello.txt")?;
  //the `?` operator can only be used in a function that returns `Result` or `Option` (or another type that implements `FromResidual`)
}
```

main可以返回任何实现了`std::process::Termination` trait的类型

```rust
use std::error::Error;
use std::fs::File;

fn main() -> Result<(), Box<dyn Error>> {
    let greeting_file = File::open("hello.txt")?;

    Ok(())
}
```

 `Box<dyn Error>` 表示任何类型的错误，所以这个main函数中可以用`?`返回任何类型的错误.

 `?` 返回Option类型时，如果时none会提前返回None，否则会返回Some中的值。

```rust
fn last_char_of_first_line(text: &str) -> Option<char> {
    text.lines().next()?.chars().last()// 返回文本的第一行的最后一个字符
}
```

### Summary

painic用在程序出现不可恢复的错误。当在开发例子程序，原型程序或测试程序时，使用unwrap或expect产生painic可以简化错误处理，提前发现错误，等后续正式的程序中进行错误处理让程序更健壮。

当程序执行的前提假设，约定或内存已经被破坏，或者使用了无效数据，这些错误程序已经无法控制，此时需要panic来提醒程序员强制处理这些问题。

result用在错误时符合预期的，但还是恢复的场景或程序还能处理，例如请求超时。

通过封装结构体，来确保数据的正确性，例如下面例子中获取一个1-100之间的数字，只要这个对象可以创建出来，它就一定满足1-100这个范围约定。

```rust
pub struct Guess {
    value: i32,
}
impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 || value > 100 {
            panic!("Guess value must be between 1 and 100, got {}.", value);
        }
        Guess { value }
    }
    pub fn value(&self) -> i32 {
        self.value
    }
}
```


