---
title: 并行与并发
date: 2024-02-23 09:36:49
categories:
- programming
tags:
- learning
---

## 并行与并发

### 基本差异

打开两个文件A和B，分别向其中写入数据后保存，实现的方式有三种模式：

* 同步顺序执行

  先打开文件A，向其中写入内容，关闭A文件，再打开文件B向其中写入内容，关闭B文件

* 多线程执行（并行）

  创建两个线程1和2，线程1中打开文件A，线程2中打开文件B，分别在两个线程中处理

* 异步IO（并发）

  在同一个线程中分派两个任务1和2，分别在1和2中执行打开文件A和文件B的操作，线程中先执行任务1，当1执行到IO操作时，转向执行任务2，任务2执行到IO操作时，线程空闲，等待系统通知，当1的IO执行完成，线程执行1的写文件程序，并再次等待1的IO操作，2也是类似的行为，直到两个任务都执行完成。

Erlang之父Joe Armstrong一个例子解释并行与并发的区别 [并发和并行 - Rust语言圣经(Rust Course)](https://course.rs/advance/concurrency-with-threads/concurrency-parallelism.html) ：

  ![concurrent](../../uploads/program/concurrent.png)

  ![concurrent](/uploads/program/concurrent.png)

* **并发(Concurrent)** ：多个队列使用同一个咖啡机，每个队列轮换着使用（未必是 1:1 轮换，也可能是其它轮换规则），最终每个人都能接到咖啡。**同时存在**轮流处理。

* **并行(Parallel)** ：每个队列都拥有一个咖啡机，最终也是每个人都能接到咖啡，但是效率更高，因为**同时**可以有两个人在接咖啡。**同时执行**。

对于单核上的多线程，其实也是一种并发，因为多个线程之间并没有真正意义上的同时执行，只是轮流执行多个线程。对于多核处理器，多个线程可以在不同的处理器上同时执行，所以是并行。可以把**并行看做是一种特殊的并发**，因为同时执行的一定同时存在。

### 并行

并行一般指多个进程或多个线程同时运行在多个处理器上，强调同时执行。

### 并发

并发并不要求必须同时执行，多个任务都是同时存在的。比并行的概念更宽泛。

#### 并发编程模型

不同语言实现并发编程的模型不尽相同：

* 操作系统线程：线程池方式让多个任务执行在多个线程上，需要处理线程同步，以及线程切换负载也很大。
* 事件驱动编程：通过事件回调机制，性能很高，但是由于回调会导致程序不是顺序执行，多层回调会导致程序很难维护，要找出哪一个回调上出的问题，代码上也会有很多回调函数套回调函数的情况。
* 协程：像线程，但是它对系统底层进行抽象，实现语言自己的类似线程模型，语言的M个线程会以N个操作系统线程执行
* actor模型：把多个并发的计算任务分割为actor，actor之间通过消息传递，类似分布式系统。

###  异步

异步执行一个任务时不需要等待它执行完成，可以直接进行别的操作。

同步必须等当前任务执行完成后，才能继续执行后续的操作。

异步和并发没有关系。异步编程更像是一种并发编程模型，它可以让大量的任务并发执行在很小数量的操作系统线程上。

例如编程书中，一般并发的章节中讲的都是多线程的知识，而异步的章节中讲的是`Future`和`async`

异步编程比使用多线程更便捷，不需要考虑线程间数据竞争和加锁的问题，代码写起来和同步执行的代码类似。

#### rust中异步

 [Why Async? - Asynchronous Programming in Rust (rust-lang.github.io)](https://rust-lang.github.io/async-book/01_getting_started/02_why_async.html) 

##### 什么时候用线程？

当任务的数量比较少时。线程会有CPU切换和内存使用，切换线程非常占用系统资源。多线程可以不用大量修改现有的同步代码，系统编程时可以调整线程的优先级，这在对于时效敏感的程序很重要。使用多线程下载两个文件伪代码

```rust
fn get_two_sites() {
    // Spawn two threads to do work.
    let thread_one = thread::spawn(|| download("https://www.foo.com"));
    let thread_two = thread::spawn(|| download("https://www.bar.com"));

    // 等待两个线程的join返回，即两个线程都执行完成
    thread_one.join().expect("thread one panicked");
    thread_two.join().expect("thread two panicked");
}
```

##### 什么时候使用异步？

程序中有大量的IO操作，例如服务器和数据库程序。以及程序的任务数量远大于操作系统的线程数时也适合用异步async，因为异步的runtime使用少量的系统线程，可以处理大量的轻量级任务。由于runtime的引入，使用异步的程序二进制文件也会大一些。实现异步时会生成异步函数的状态机代码，导致程序变大。

异步并不比多线程好，它只是另一种方案。如果没有大量计算场景，不需要使用异步，多线程更简单。

使用异步下载两个文件伪代码示例

```rust
async fn get_two_sites_async() {
    // Create two different "futures" which, when run to completion,
    // will asynchronously download the webpages.
    let future_one = download_async("https://www.foo.com");
    let future_two = download_async("https://www.bar.com");

    // 执行两个future，直到两个都执行完成
    join!(future_one, future_two);
}
```

##### rust异步编程模型

 [Rust Runtime 设计与实现-科普篇 | 下一站 - Ihcblog!](https://www.ihcblog.com/rust-runtime-design-1/#more) 

rust中的异步主要用runtime来控制任务的调度执行，语言自身并没有runtime的实现，需要自己实现，tokio就有自己的runtime。

一个runtime有三个部分：

* Executor 负责任务调度，并执行相关操作
* Reactor 与操作系统的实际机制`epoll`交互，当系统通知某个事件发生后，它通过Waker通知Executor对应的任务可以执行了
* 任务队列 可以想象为有两个队列，一个是正在执行的队列，一个是等待唤醒的队列，这两个队列都由Executor来控制调度