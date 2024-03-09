---
title:  Rust Web Server
date: 2024-03-09 09:42:49
categories:
- programming
tags:
- rust
- learning
---

## Rust Web Server

### TCP连接

#### 监听

`TcpListener` 用来监听Tcp的连接，他的`incoming()`返回的`TcpStream`表示了一个tcp连接。通过遍历这个stream可以获取客户端发来的数据，并进行应答。当stream执行出循环体后，就会断开这个连接，下面的例子种一个循环对应一个连接。

```rust
let listener = TcpListener::bind("127.0.0.1:7878").unwrap()
for stream in listener.incoming() {
    let stream = stream.unwrap();
    println!("new connection established");
}
```

端口号在1204以下需要管理员权限，这里7878是rust四个字母在手机的9宫格按键。

运行程序后，直接在浏览器访问`http://127.0.0.1:7878/`会得到`The connection was reset.`的提示。程序的控制台实际上已经输出了很多次`new connection established`。之所以有多次请求是因为浏览器还在请求其他的网站数据，例如icon等。

在浏览器的控制台可以看到有很多次请求，也就建立了多次连接，每一次服务端执行出循环，这个连接就被drop了。

#### 处理请求

使用`BufReader`来包装一个stream的可变引用，它提供了buffer机制方便读取数据，例如下面的`lines()`方法。

```rust
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let http_request: Vec<_> = buf_reader.lines()
            .map(|result| result.unwrap()) // 得到每一行的字串
            .take_while(|line| !line.is_empty()) // 剔除其中的空字串
            .collect();
    println!("Request: {:?}", http_request);
}
```

控制台会输出浏览器的请求。

```shell
new connection established
Request: ["GET / HTTP/1.1", "Host: 127.0.0.1:7878", "Connection: keep-alive", "Cache-Control: max-age=0", "sec-ch-ua: \"Chromium\";v=\"122\", \"Not(A:Brand\";v=\"24\", \"Google Chrome\";v=\"122\"", "sec-ch-ua-mobile: ?0", "sec-ch-ua-platform: \"Windows\"", "Upgrade-Insecure-Requests: 1", "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36", "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7", "Sec-Fetch-Site: cross-site", "Sec-Fetch-Mode: navigate", "Sec-Fetch-User: ?1", "Sec-Fetch-Dest: document", "Accept-Encoding: gzip, deflate, br, zstd", "Accept-Language: en-US,en;q=0.9,zh-CN;q=0.8,zh;q=0.7"]
```

### HTTP协议

http是超文本传输协议，它的请求都是文本类型。

#### 请求协议

```shell
Method Request-URI HTTP-Version CRLF ---> "GET / HTTP/1.1"
headers CRLF ---> "Host: 127.0.0.1:7878"之后都是请求头
message-body Get请求没有消息体
```

#### 应答协议

应答和请求类似

```shell
HTTP-Version Status-Code Reason-Phrase CRLF
headers CRLF  这里定义多长Content-Length的内容，浏览器就只会接收多少内容
message-body  实际的内容
```

通过读取一个文件`index.html`应答给客户端，按照协议把三行信息通过`stream.write_all()`应答给客户端

```rust
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let http_request: Vec<_> = buf_reader.lines()
            .map(|result| result.unwrap()) // 得到每一行的字串
            .take_while(|line| !line.is_empty()) // 剔除其中的空字串
            .collect();
    println!("Request: {:?}", http_request);

    let status_line = "HTTP/1.1 200 OK";
    let contents = fs::read_to_string("index.html").unwrap();
    let length = contents.len();
    let response = format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");
    stream.write_all(response.as_bytes()).unwrap();
}
```

#### 处理请求不同地址

http请求`"GET / HTTP/1.1"`中的第2段表示了请求的地址，因此根据不同的请求地址可以转到不同的应答处理函数。这里可以简单将非`/`根目录的请求都应答为404.

```rust
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    // 只获取请求的方法和地址，即 "GET / HTTP/1.1"
    let http_request = buf_reader.lines().next().unwrap().unwrap();
    println!("Request: {:?}", http_request); // Request: "GET / HTTP/1.1"

    let (status_line, filename) = if http_request == "GET / HTTP/1.1" {
        ("HTTP/1.1 200 OK", "index.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND", "404.html")
    };
    
    let contents = fs::read_to_string(filename).unwrap();
    let length = contents.len();
    let response = format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");
    stream.write_all(response.as_bytes()).unwrap();
}
```



### 使用线程池处理多个请求

每当有一个新任务时，可以从线程池中取出一个线程执行这个任务。线程池中通过一个队列处理所有收到的请求，它最多并发执行线程池大小的任务。使用线程池是最简单的方案，还可以有`fork/join模型`,`单线程的异步IO`以及`多线程的异步IO`。

单独创建一个`src/lib.rs`来存放线程池实现代码，这样这个库以后还可以被其他应用程序使用

```rust
use std::{sync::{mpsc, Arc, Mutex}, thread};
// 用来包装一个线程
struct Worker {
  	// 每一个worker都有一个自己的id用来区分不同的worker
    id: usize,
    // thread::spawn的返回值是JoinHandle<T>
    thread: thread::JoinHandle<()>,
}

impl Worker {
    fn new(id:usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || 
        loop {
            // 循环一直等待接收任务，recv是一个阻塞调用，当它收到新的消息后，才会继续执行
            let job = receiver.lock().unwrap().recv().unwrap();
            println!("Worker {id} got a job; executing.");
            // 执行闭包
            job();
        });
        Worker { id, thread }
    }
}
// 表示一个闭包函数
type Job = Box<dyn FnOnce() + Send + 'static>;

pub struct ThreadPool {   
  	// 线程池中有多个worker
    workers: Vec<Worker>,
  // 用于给各个worker通知的sender
    sender: mpsc::Sender<Job>,
}
// 使用cargo doc --open 就可查看当前代码的文档
impl ThreadPool {
    /// Create a new ThreadPool.
    ///
    /// The size is the number of threads in the pool.
    ///
    /// # Panics
    ///
    /// The `new` function will panic if the size is zero.
    pub fn new(size: usize) -> ThreadPool{
        assert!(size > 0);
        // 通过channel把传给线程池的闭包传递给各个子线程
        let (sender, receiver) = mpsc::channel();
        // 一个生产者，多个消费者接收任务，Mutex保证一次只有一个线程能获取到这个消息
        let receiver = Arc::new(Mutex::new(receiver));
        // 提前申请好使用的内存空间，效率更高
        let mut workers = Vec::with_capacity(size);
		// 创建多个worker
        for id in 0..size {
          	// Arc::clone 让多个线程都能引用这个receiver
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }
        ThreadPool { workers, sender }
    }

  	/// 线程池的执行函数
    pub fn execute<F>(&self, f: F)
    where 
        F: FnOnce() + Send + 'static,
    {
        // 把闭包函数包成一个对象
        let job = Box::new(f);
        // 把闭包函数发送给worker执行，哪个worker收到就执行它
        self.sender.send(job).unwrap();
    }
}
```

在main.rs文件中使用这个线程池，首先要引入进来`use webserver::ThreadPool;`

```rust
use webserver::ThreadPool;

fn start_server() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
	// 创建5个线程的线程池
    let pool = ThreadPool::new(5);
    for stream in listener.incoming() {
        let stream = stream.unwrap();
        println!("new connection established");
      	// 统一让线程池来处理
        pool.execute(|| {
            handle_connection(stream);
        });        
    }
}
```

需要特别注意的是Worker中的循环写法使用了loop，而不是while

```rust
let thread = thread::spawn(move || {
    while let Ok(job) = receiver.lock().unwrap().recv() {
        println!("Worker {id} got a job; executing.");
        job();
    }
});
```

如果使用了while，receiver.lock()的声明周期在while循环体这一次的执行完成后，才能释放，也就是锁也会在job()执行完成后才能释放，导致其他线程在这个job没有执行完前都不能获取锁，也就不能同通道中获取新的任务信息，其实就没有多线程执行的效果了，因为其他线程获取`receiver.lock().unwrap().recv()`这个操作被正在执行任务的这个线程的lock阻塞了。而使用let的方式，=右边的表达式在let执行完后，就会被释放了，锁的释放在执行Job之前，所以如果job耗时也不会影响其他线程拿锁。