---
title: Rust Learning-Threads
date: 2024-01-07 09:42:49
categories:
- programming
tags:
- rust
- learning
---

## RUST Threads 

 [Fearless Concurrency](https://doc.rust-lang.org/stable/book/ch16-00-concurrency.html#fearless-concurrency)

多线程的常见问题：

* 条件竞争：多个线程同时访问同一个数据或资源
* 死锁：两个线程互相等待另一个线程执行结束后，再继续执行自己
* 一些特殊场景下业务相关的偶发故障

### 基本用法

rust标准库创建的线程数量和操作系统实际创建的线程数量是`1:1`的，即一个程序在rust创建了多少个线程，操作系统实际就创建了多少个线程。

#### 创建线程
使用`thread::spawn()`创建一个线程，传入的闭包中执行子线程执行的代码。当主线程结束时，所有的子线程将会被强制结束执行。例如下面的子线程只执行到19左右。

```rust
use std::thread;
use std::time::Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..50 {
            println!("spawned thread goes to {i} ***");
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..20 {
        println!("main thread goes to {i} ###");
        thread::sleep(Duration::from_millis(1));
    }
    println!("Main thread run out"); 
}
```

#### 线程等待

通过 `thread::spawn` 的返回值 `JoinHandle`可以控制线程调度。当调用  `JoinHandle`的`join`方法时，它会阻塞当前调用它的线程，直到它指向的线程执行结束后，才返回给当前调用线程继续执行，可以想象为一个红灯，当子线程内容执行完后，它会切换为绿灯。

```rust
fn main() {
    let handle = thread::spawn(|| {
        for i in 1..50 {
            println!("spawned thread goes to {i} ***");
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..20 {
        println!("main thread goes to {i} ###");
        thread::sleep(Duration::from_millis(1));
    }

    handle.join().unwrap();

    println!("Main thread run out"); 
}
```

现在子线程可以执行输出到49了。

#### move环境数据

在子线程中使用它上下文环境中的数据需要获取数据的所有权，此时需要在闭包前加上`move`。这样数据被子线程获取所有权，外部线程在使用它时会编译错误，也就不会出现子线程使用过程中外部已经把数据修改了的问题。

```rust
use std::thread;

fn main() {
    let mut v = vec![1, 2, 3];

    let handle = thread::spawn(move || {
        println!("Here's a vector: {:?}", v);
        for i in &mut v {
            *i += 10;
        }
        println!("the vector {:?}", v);
    });    

    handle.join().unwrap();

    //println!("the vector {:?}", v); //borrow of moved value: `v`
}
```

### 消息传递

现在流行线程间传输数据使用消息方式，而不是使用共享内存。Go语言提倡不要使用共享内存来通信消息，相反要用通信消息来共享内存数据。 [the Go language documentation](https://golang.org/doc/effective_go.html#concurrency): “Do not communicate by sharing memory; instead, share memory by communicating.”

rust使用通道(channel)的机制传递消息。可以把通道看作定向的河流，一个数据可以通过河流从发送者传递给接收者。当发送者或接收者任何一方销毁，这个通道就关闭了。

使用`mpsc::channel`来创建一个通道，它返回一个元组，元组的第一个元素时发送者(*transmitter*)，第二个元素时接收者( *receiver* )。 `mpsc` 是 *multiple producer, single consumer*的缩写。

发送者有个`send`方法，它以发送的数据作为参数，返回一个 `Result<T, E>`，如果接收者已经被释放或没有发数据的目标地方，`send`会返回错误。

接收者有个`recv`方法，它会阻塞当前的线程执行直到一个数据通过通道发送，然后`recv`返回 `Result<T, E>`。当传输者释放，`recv`会返回一个错误信号。

`try_recv`不会阻塞当前的线程，会立即返回一个 `Result<T, E>`。如果当前有收到数据会得到一个`Ok`否则得到`Err`。可以通过循环调用`try_recv`来实现在等待数据的时候，在当前线程做一些别的事情，例如1s收一次数据，在1s间隔中等待下一次检查数据前可以做一些其他计算。

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
        //println!("val is {}", val); // error borrow of moved value: `val`
    });

    let received = rx.recv().unwrap(); // blocked until received data
    println!("Got: {}", received); // Got: hi
}
```

子线程中被发送出去的数据已经被**move**走了，所以子线程中不能再使用这个数据，从而保证了多线程数据访问安全。这些错误rust在编译期就能识别出来，运行时错误。

可以通过**迭代器**循环接收数据。下例中发送者每秒发送一个数据，接收者迭代器每收到一个数据执行一次，直到通道被关闭，迭代器才会结束。

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));            
        }        
    });

    for received in rx {
        println!("Got: {}", received);
    }
}
```

猜数字例子使用多线程，在一个线程中获取输入，另一个线程中打印输入的数据

```rust
use std::sync::mpsc;
use std::thread;
use std::io;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        loop {
            println!("Input your guess: ");    
            let mut guess = String::new();  // mut 可变变量
            io::stdin()
                .read_line(&mut guess)
                .expect("Failed to read line");
    
            let guess: u32 = match guess.trim().parse() {
                Ok(num) => num,
                Err(_) => continue, // -是一个通配符，匹配所有Err值，如果不能转换为数字，进入下次循环
            };
    
            if guess == 0 {
                break;
            }
            tx.send(guess).unwrap();        
        }      
    });

    for received in rx {
        println!("You guessed: {received}"); // {}占位符，可以打印变量或表达式结果
    }
}
```

通过`clone`方法可以实现多个生产者，即多个发送者一个接收者. 克隆出来的对象也可以给通道的接收者发送数据。

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();

    let tx1 = tx.clone(); // 克隆一个发送者
    thread::spawn(move || {
        let vals = vec![
            String::from("1"),
            String::from("1"),
        ];

        for val in vals {
            tx1.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });

    thread::spawn(move || {
        let vals = vec![
            String::from("2"),
            String::from("2"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });

    for received in rx {
        println!("Got: {}", received);
    }
}
```

### 共享内存

使用通道的方式传递数据时，发送的数据发出去后，发送者不能再使用这个数据。共享内存的方式允许多个线程访问访问同一个数据。这时需要使用**Mutex**(mutual exclusion)互斥量。它可以限制一个数据当前只被一个线程使用，类似多人聊天房间抢麦，当一个人想要发言，他要先申请麦的权限，当他获取到麦后，可以讲话，他讲完后，必须把麦释放给下一个人。使用Mutex需要注意两点：

* 使用数据前，需要请求锁
* 使用完数据后，需要释放锁

使用 `Mutex<T>` 的`new`方法创建一个 `Mutex<T>` 对象，使用`lock`方法来请求锁。`lock`方法会阻塞当前线程，直到获取到锁。 `Mutex<T>` 是一个智能指针，`lock`会返回一个`MutexGuard`对象，`MutexGuard`实现了`Deref`来获取内部数据，同时实现了`Drop`在退出作用域时可以释放锁。

 `Mutex<T>` 提供了内部可变性，虽然`let counter = Arc::new(Mutex::new(0));`的counter不是可变的，但是通过 `Mutex<T>`可以修改其内部数据。

由于通过move把counter的所有权移入了子线程中，当有多个子线程时，每个线程都要获取counter的所有权，此时需要使用`Rc<T>`来创建一个引用计数的值，让多个线程都可以拥有一个数据，但是`Rc<T>`不是线程安全的，因为它要在内部对引用计数进行增加或减少，而多个线程可能同时操作不同，因此需要使用`Arc<T>`一个提供原子性的计数器*atomically reference counted* ，可以用来在多个线程中获取多个所有权。

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter); // 获取一个引用计数，以便在多个线程中都可以使用
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap(); // 获取可变数据
            println!("run in sub threads: {}", num);
            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles { // 等所有线程执行结束
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

### Sync和Send Trait

rust语言自身提供了很少并发特性。大部分机制都通过std或其他crate的方式支持。

Sync和Send这两个Trait是语言核心提供语法。

所有实现了Send的Trait的对象可以在多个对象之间传递，这些对象是线程安全的。所有的基本数据类型都是支持Send的，其他数据类型默认不是Send主要为了性能。

实现了Sync的Trait的对象可以被多个线程引用。一个不可变引用&T是支持Send的，那么类型T就是Sync的，因为它的引用可以被传递给其他线程，多个线程就能引用它。基本数据类型是Sync的，Mutex<T>`也是Sync的。

完全由支持Send和Sync的类型组成的新类型也是Send和Sync的，所以一般不用自己手动实现Send和Sync，他们也没有需要实现的方法


