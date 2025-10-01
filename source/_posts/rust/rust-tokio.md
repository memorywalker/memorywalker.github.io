---
title: Rust的Tokio库
date: 2025-10-01T14:53:00
categories:
  - rust
tags:
  - rust
  - tokio
---
## Tokio

[官网地址](https://tokio.rs/)

[教程地址](https://tokio.rs/tokio/tutorial/) 这个教程实现了简单的redis服务端和客户端。

Tokio是rust语言的一个异步运行时，它包括以下组件：
* 执行异步代码的多线程运行时
* 标准库的异步版本
* 大量的库生态系统，基于它有许多子库项目

#### 什么情况不需要Tokio？

* rust主要用于IO密集的应用，对于CPU密集的应用不适用，这种情况下可以用`rayon`
* 读取大量文件，相对于线程池的方法tokio没有什么优势，操作系统底层没有提供异步文件访问的API
* 发送一个web请求，tokio主要解决同时做多件事情的场景，对于请求比较少的情况，可以简单的使用同步执行程序。

### 异步编程

大部分的程序代码安装它书写的顺序逐行执行，同步执行程序中，当遇到一个耗时操作时，代码的执行会阻塞直到这个耗时操作执行完成，再执行下面的操作(代码语句)。例如建立一个网络连接，程序都会等待连接建立完成后，再执行后面的语句。

异步编程中，对于耗时操作会被挂起到后台，但是当前的线程不会被阻塞，后面的代码还可以正常继续执行，一旦耗时操作完成，被挂起的操作可以被继续执行。异步编程可以提高程序的执行效率，但是程序也会更复杂，因为需要再耗时任务完成时，恢复之前的操作和状态。
#### rust中的异步编程

函数名称中使用`async`修饰符告诉编译器，这个函数执行异步操作，编译器在编译时把这个函数编译为异步运行的例程(routine)。

在`async fn`作用域内调用`.await`函数都会把当前执行切回当前线程中，以执行当前操作的后续代码。
调用异步函数时，它的函数体不会立即执行，而是立即返回一个代表这个操作的值，类似一个0个参数的闭包，它的类型是实现了`Future`trait的一个异步类型，需要在这个返回值上执行`.await`操作才能执行函数体。

```rust
async fn say_world() {
    println!("world");
}  

#[tokio::main]
async fn main() {
    // `say_world()` 现在还不会执行它的函数体.
    let op = say_world();
    // This println! comes first
    println!("hello"); 
    // 对返回值`op`调用 `.await` 才开始执行函数体.
    op.await;
}
```

一个异步函数必须在一个运行时中执行，这个运行时中实现了异步任务调度，事件IO，定时器等。运行时不会自动运行，所以需要main函数启动它。
 `#[tokio::main]`是一个宏，它可以把`async fn main()`转换为同步`fn main()`，并在其中初始化一个运行时实例并执行异步的main函数。
 
```rust
#[tokio::main] 
async fn main() {
	println!("hello"); 
}
```
等价于
```rust
fn main() {
	// 创建一个运行时
    let mut rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async { // 在运行时中运行异步代码
        println!("hello");
    })
}
```

### 并发(Concurrency)

并发：一个人同时做两个工作，并在这两个工作上进行切换
并行：两个人各自负责一个工作

Tokio可以在一个线程中并发的执行多个任务，而不用像通常的创建多个线程并行的处理任务。
**绿色线程(Green Thread)** 在用户层通过一个运行时或虚拟机调度和管理的线程，而不是调用操作系统底层的线程。
一个Tokio中的任务是一个异步绿色线程，通过给`tokio::spawn`传入一个`async`修饰的代码块来创建，`tokio::spawn`返回的 `JoinHandle`可以让外部和任务进行交互。外部程序代码可以通过返回值 `JoinHandle`上调用`.await`来获取任务块内部的返回值。

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        // Do some async work
        "return value"
    });  

    // Do some other work  
	// 对任务的返回值调用await获取代码块的返回值
    let out = handle.await.unwrap();
    println!("GOT {}", out);
}
```

#### 任务

任务是Tokio的调度器管理的执行单元，创建一个任务就是把它提交给Tokio的调度器。
创建的任务可能运行在创建它的线程中，也有可能运行在一个不同的运行时所在的线程中。任务在创建后也可以在不同的线程中移动。
Tokio中的任务非常轻量级，它只需要64字节的内存，所以应用可以放心的创建和使用任务。

Tokio的任务的**类型的生命周期**是`'static`，因此创建的任务代码中不能引用任务之外的数据。如下代码会报错`error[E0373]: async block may outlive the current function, but it borrows `v`, which is owned by the current function`

```rust
#[tokio::main]
async fn main() {
    let v = vec![1, 2, 3];  

    task::spawn(async {
        println!("Here's a vec: {:?}", v);
    });
}
```

因为变量`v`并没有被move到异步代码块中，它的所有权还在main函数中。按照编译器的提示需要在`task::spawn(async move {`加入move关键字，从而把变量`v`移入异步代码块中。如果一个数据被多个任务访问使用，可以使用`Arc`类型，共享数据。

Tokio创建的任务必须实现`Send`，这样Tokio运行时可以把挂起的任务可以在不同的线程间移动。
当`.await`被调用时，任务被暂停挂起，当前的执行权转移给了调度器，当任务下一次被执行，它从上次暂停的位置恢复。所以所有`.await`之后使用的状态必须在任务中保存，如果这些状态是可以`Send`，这个任务就可以在不同的线程间移动，反之如果状态不能`Sned`，任务也就不能在多个线程间移动。以下代码会报错
```rust
#[tokio::main]
async fn main() {
    tokio::spawn(async {
        let rc = Rc::new("hello");
        // `rc` is used after `.await`. It must be persisted to
        // the task's state.
        yield_now().await;
        println!("{}", rc);
    });
}
```

#### 服务端完整代码

```rust
use tokio::net::{TcpStream, TcpListener};
use mini_redis::{Connection, Frame};

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    loop {
        let (socket, _) = listener.accept().await.unwrap();
        // 一个socket连接一个task，socket对象需要被moved到任务中被执行
        tokio::spawn(async move {
            process(socket).await;
        });
    }
}

async fn process(socket: TcpStream) {
    use mini_redis::Command::{self, Get, Set};
    use std::collections::HashMap;

    // A hashmap is used to store data
    let mut db = HashMap::new();

    // 处理一个连接
    let mut connection = Connection::new(socket);

    // Use `read_frame` 解析请求的命令
    while let Some(frame) = connection.read_frame().await.unwrap() {
        let response = match Command::from_frame(frame).unwrap() {
            Set(cmd) => {
                // The value is stored as `Vec<u8>`
                db.insert(cmd.key().to_string(), cmd.value().to_vec());
                Frame::Simple("OK".to_string())
            }
            Get(cmd) => {
                if let Some(value) = db.get(cmd.key()) {
                    // `Frame::Bulk` expects data to be of type `Bytes`. This
                    // type will be covered later in the tutorial. For now,
                    // `&Vec<u8>` is converted to `Bytes` using `into()`.
                    Frame::Bulk(value.clone().into())
                } else {
                    Frame::Null
                }
            }
            cmd => panic!("unimplemented {:?}", cmd),
        };

        // 客户端应答
        connection.write_frame(&response).await.unwrap();
    }
}
```