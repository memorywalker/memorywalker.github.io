---
title: Rust异步编程
date: 2026-04-02T23:58:00
categories:
  - rust
tags:
  - rust
  - async
  - 并行
---
## Rust异步编程


当一个线程被I/O阻塞，只能等系统I/O调用执行完成后，这段时间存在资源浪费。
例如处理n多个客户端请求，对于每个请求创建一个线程，而每个线程的栈可能有上千字节，在线程等待期间，就会有上千字节内存被占用，但是又不做任何其他事情，当请求的数量巨大时，内存的占用就非常明显了。

以下内容是AI生成的解释

## 一、Rust 异步编程核心概念

### 1.1 异步编程的本质

Rust 的异步编程基于 **零成本抽象** 的理念，在编译期将 `async/await` 语法转换为状态机，避免了运行时的额外开销。与 Go 的 goroutine 或 Java 的虚拟线程不同，Rust 的异步模型是 **惰性求值（lazy）** 的——创建一个 Future 并不会立即执行，需要 executor 来驱动它。

### 1.2 核心三要素

```
┌─────────────────────────────────────────────┐
│              Rust 异步编程架构              │
├─────────────────────────────────────────────┤
│                                             │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│   │  Future  │  │  Waker   │  │ Executor │  │
│   │  (任务)  │  │ (唤醒器) │  │ (执行器) │  │
│   └─────┬────┘  └─────┬────┘  └─────┬────┘  │
│         │             │             │       │
│         └─────────────┼─────────────┘       │
│                       │                     │
│              Poll 驱动模型                  │
│                                             │
└─────────────────────────────────────────────┘
```

### 1.3 Future Trait

`Future` 是 Rust 异步编程的基石：

```rust
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),    // 任务完成，返回结果
    Pending,     // 任务未完成，稍后再试
}
```

**工作流程：**

1. Executor 调用 `Future::poll()` 尝试推进任务
2. 如果返回 `Poll::Ready(value)`，任务完成
3. 如果返回 `Poll::Pending`，Future 通过 `Waker` 注册回调
4. 当外部事件就绪时（如 I/O 完成），`Waker::wake()` 通知 Executor 重新 poll

### 1.4 async/await 的编译期转换

```rust
// 你写的代码：
async fn fetch_data() -> String {
    let response = make_request().await;
    let body = read_body(response).await;
    body
}
```

编译器会将其转换为类似如下的状态机：

```rust
enum FetchDataState {
    State0 { /* 初始状态 */ },
    State1 { request_future: MakeRequestFuture },
    State2 { read_future: ReadBodyFuture },
    Completed,
}

struct FetchDataFuture {
    state: FetchDataState,
}

impl Future for FetchDataFuture {
    type Output = String;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<String> {
        loop {
            match self.state {
                State0 => {
                    // 创建 request future，转到 State1
                    let fut = make_request();
                    self.state = State1 { request_future: fut };
                }
                State1 { ref mut request_future } => {
                    match Pin::new(request_future).poll(cx) {// poll()的第一个参数是self: Pin<&mut Self>，所以要把request_future用Pin包起来 
                        Poll::Ready(response) => {
                            let fut = read_body(response);
                            self.state = State2 { read_future: fut };// 当前异步去到了数据，切换到下一个状态
                        }
                        Poll::Pending => return Poll::Pending,
                    }
                }
                State2 { ref mut read_future } => {
                    match Pin::new(read_future).poll(cx) {
                        Poll::Ready(body) => {
                            self.state = Completed;// 切换到最终的完成状态
                            return Poll::Ready(body);
                        }
                        Poll::Pending => return Poll::Pending,
                    }
                }
                Completed => panic!("polled after completion"),
            }
        }
    }
}
```

### 1.5 Pin 与自引用

`Pin` 的存在是为了解决 **自引用结构体** 问题。当 async 块跨 await 点持有引用时，编译器生成的状态机会包含自引用字段：

```rust
async fn example() {
    let data = vec![1, 2, 3];
    let reference = &data;  // 自引用：reference 指向同一结构体中的 data
    some_async_op().await;   // await 点 —— 由于reference在await之后还有使用，所以状态机需要保存 data 和 reference
    println!("{:?}", reference);
}
```

`Pin<&mut Self>` 确保 Future 在内存中不会被移动，从而保证自引用的安全性。

### 1.6 Waker 机制

`Waker` 是连接 **I/O 事件系统** 和 **Executor** 的桥梁：

```
┌──────────┐     poll()       ┌──────────┐
│ Executor │ ───────────────> │  Future  │
│          │ <─────────────── │          │
│          │   Pending +      │          │
│          │   保存 Waker     │          │
└────┬─────┘                  └──────────┘
     │                              │
     │  wake()                      │ 注册到 I/O 系统
     │ <────────────────────────────┘
     │
     │  重新 poll()
     └──────────────────────────────>
```

---

## 二、手写一个简单的异步运行时

下面我们从零开始构建一个包含以下组件的异步运行时：

- **Task（任务）**：封装 Future
- **Executor（执行器）**：调度和执行任务
- **Spawner（生成器）**：向 Executor 提交新任务
- **简单的定时器 Future**：演示自定义 Future 的实现
- **简单的异步 TCP 请求**：演示网络 I/O

### 完整代码

```rust
use std::{
    collections::HashMap,
    future::Future,
    io::{Read, Write},
    net::TcpStream,
    pin::Pin,
    sync::{
        mpsc::{self, Receiver, Sender},
        Arc, Mutex,
    },
    task::{Context, Poll, RawWaker, RawWakerVTable, Waker},
    thread,
    time::{Duration, Instant},
};

// ============================================================
// 第一部分：Task 定义
// ============================================================

/// 一个可被 Executor 调度的异步任务
struct Task {
    /// 被包装的 Future，使用 Pin<Box<>> 确保不会被移动
    future: Pin<Box<dyn Future<Output = ()> + Send>>,
    /// 任务 ID（用于调试）
    id: usize,
}

// ============================================================
// 第二部分：Executor（执行器）和 Spawner（任务生成器）
// ============================================================

/// 任务生成器：用于向 Executor 提交新的异步任务
#[derive(Clone)]
struct Spawner {
    sender: Sender<Task>,
}

impl Spawner {
    /// 生成一个新的异步任务
    fn spawn<F>(&self, id: usize, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        let task = Task {
            future: Box::pin(future),
            id,
        };
        self.sender
            .send(task)
            .expect("任务队列已满或 Executor 已关闭");
        println!("[Spawner] 已提交任务 #{}", id);
    }
}

/// 执行器：负责驱动所有异步任务直到完成
struct Executor {
    /// 就绪队列：存放等待执行的任务
    ready_queue: Receiver<Task>,
    /// 用于重新调度任务的发送端
    sender: Sender<Task>,
}

impl Executor {
    /// 创建一对 (Executor, Spawner)
    fn new() -> (Self, Spawner) {
        let (sender, receiver) = mpsc::channel();
        let spawner = Spawner {
            sender: sender.clone(),
        };
        let executor = Executor {
            ready_queue: receiver,
            sender,
        };
        (executor, spawner)
    }

    /// 运行所有任务直到全部完成
    fn run(&self) {
        println!("[Executor] 开始运行...\n");

        // 持续从队列中取出任务并 poll
        while let Ok(mut task) = self.ready_queue.recv() {
            let task_id = task.id;
            println!("[Executor] 正在 poll 任务 #{}...", task_id);

            // 为这个任务创建一个 Waker
            let waker = create_waker(task_id, self.sender.clone());
            let mut cx = Context::from_waker(&waker);

            // 调用 Future::poll
            match task.future.as_mut().poll(&mut cx) {
                Poll::Ready(()) => {
                    println!("[Executor] ✅ 任务 #{} 已完成!\n", task_id);
                }
                Poll::Pending => {
                    println!("[Executor] ⏳ 任务 #{} 返回 Pending，等待唤醒...\n", task_id);
                    // 注意：这里我们不重新入队
                    // 任务会在 Waker::wake() 被调用时重新入队
                    // 但为了简化，某些 Future 内部会自行处理重新调度
                    // 在我们的实现中，Waker 会将任务重新发送到队列

                    // 我们需要保存任务，以便 Waker 能重新调度它
                    // 在简化版本中，我们通过闭包捕获来实现
                    // 这里将任务发送到一个等待区域
                    PENDING_TASKS
                        .lock()
                        .unwrap()
                        .insert(task_id, task);
                }
            }
        }

        println!("[Executor] 所有任务已完成，退出。");
    }
}

// 全局 pending 任务存储（简化实现）
lazy_static::lazy_static! {
    static ref PENDING_TASKS: Mutex<HashMap<usize, Task>> = Mutex::new(HashMap::new());
}

// ============================================================
// 第三部分：Waker 的创建
// ============================================================

/// 用于存储 Waker 关联数据的结构
struct WakerData {
    task_id: usize,
    sender: Sender<Task>,
}

/// 创建一个 Waker
fn create_waker(task_id: usize, sender: Sender<Task>) -> Waker {
    let data = Arc::new(WakerData { task_id, sender });
    let raw = Arc::into_raw(data) as *const ();

    let vtable = &RawWakerVTable::new(
        waker_clone,
        waker_wake,
        waker_wake_by_ref,
        waker_drop,
    );

    unsafe { Waker::from_raw(RawWaker::new(raw, vtable)) }
}

unsafe fn waker_clone(ptr: *const ()) -> RawWaker {
    let arc = Arc::from_raw(ptr as *const WakerData);
    let cloned = arc.clone();
    std::mem::forget(arc); // 不减少原来的引用计数
    let raw = Arc::into_raw(cloned) as *const ();
    RawWaker::new(
        raw,
        &RawWakerVTable::new(waker_clone, waker_wake, waker_wake_by_ref, waker_drop),
    )
}

unsafe fn waker_wake(ptr: *const ()) {
    let arc = Arc::from_raw(ptr as *const WakerData);
    do_wake(&arc);
    // arc 在这里被 drop，引用计数减少
}

unsafe fn waker_wake_by_ref(ptr: *const ()) {
    let arc = Arc::from_raw(ptr as *const WakerData);
    do_wake(&arc);
    std::mem::forget(arc); // 不减少引用计数
}

unsafe fn waker_drop(ptr: *const ()) {
    drop(Arc::from_raw(ptr as *const WakerData));
}

fn do_wake(data: &WakerData) {
    let task_id = data.task_id;
    println!("[Waker] 唤醒任务 #{}!", task_id);

    // 从 pending 任务表中取出任务，重新发送到就绪队列
    if let Some(task) = PENDING_TASKS.lock().unwrap().remove(&task_id) {
        let _ = data.sender.send(task);
    }
}

// ============================================================
// 第四部分：自定义 Future 实现
// ============================================================

// ---------- 4.1 TimerFuture：异步定时器 ----------

struct TimerFuture {
    /// 到期时间
    target_time: Instant,
    /// 是否已经启动了后台线程
    started: bool,
    /// 共享状态：标记是否已完成
    completed: Arc<Mutex<bool>>,
}

impl TimerFuture {
    fn new(duration: Duration) -> Self {
        TimerFuture {
            target_time: Instant::now() + duration,
            started: false,
            completed: Arc::new(Mutex::new(false)),
        }
    }
}

impl Future for TimerFuture {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        // 检查是否已完成
        if *self.completed.lock().unwrap() {
            return Poll::Ready(());
        }

        // 如果还没有启动后台线程，就启动一个
        if !self.started {
            self.started = true;
            let waker = cx.waker().clone();
            let target = self.target_time;
            let completed = self.completed.clone();

            thread::spawn(move || {
                let now = Instant::now();
                if now < target {
                    thread::sleep(target - now);
                }
                *completed.lock().unwrap() = true;
                waker.wake(); // 唤醒 Executor 重新 poll 这个任务
            });
        }

        Poll::Pending
    }
}

// ---------- 4.2 HttpFuture：异步 HTTP GET 请求 ----------

struct HttpFuture {
    url: String,
    host: String,
    path: String,
    port: u16,
    started: bool,
    result: Arc<Mutex<Option<String>>>,
}

impl HttpFuture {
    /// 创建一个简单的 HTTP GET Future
    /// 参数格式：host, path, port
    fn new(host: &str, path: &str, port: u16) -> Self {
        HttpFuture {
            url: format!("{}:{}{}", host, port, path),
            host: host.to_string(),
            path: path.to_string(),
            port,
            started: false,
            result: Arc::new(Mutex::new(None)),
        }
    }
}

impl Future for HttpFuture {
    type Output = String;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<String> {
        // 检查是否已有结果
        if let Some(result) = self.result.lock().unwrap().take() {
            return Poll::Ready(result);
        }

        // 首次 poll 时启动后台 I/O 线程
        if !self.started {
            self.started = true;

            let host = self.host.clone();
            let path = self.path.clone();
            let port = self.port;
            let url = self.url.clone();
            let result = self.result.clone();
            let waker = cx.waker().clone();

            thread::spawn(move || {
                println!("  [HTTP] 开始请求: {}", url);

                let response = do_http_get(&host, &path, port);

                *result.lock().unwrap() = Some(response);
                waker.wake(); // 通知 Executor 结果已就绪
            });
        }

        Poll::Pending
    }
}

/// 执行同步 HTTP GET 请求（用于后台线程）
fn do_http_get(host: &str, path: &str, port: u16) -> String {
    let addr = format!("{}:{}", host, port);

    match TcpStream::connect_timeout(
        &addr.parse().unwrap_or_else(|_| {
            // 如果是域名，先做 DNS 解析
            use std::net::ToSocketAddrs;
            format!("{}:{}", host, port)
                .to_socket_addrs()
                .ok()
                .and_then(|mut addrs| addrs.next())
                .unwrap_or_else(|| "127.0.0.1:80".parse().unwrap())
        }),
        Duration::from_secs(5),
    ) {
        Ok(mut stream) => {
            let request = format!(
                "GET {} HTTP/1.1\r\nHost: {}\r\nConnection: close\r\n\r\n",
                path, host
            );

            stream.write_all(request.as_bytes()).ok();
            stream
                .set_read_timeout(Some(Duration::from_secs(5)))
                .ok();

            let mut response = String::new();
            stream.read_to_string(&mut response).ok();

            // 只返回前 200 个字符作为摘要
            let summary: String = response.chars().take(200).collect();
            format!("[响应摘要] {}", summary)
        }
        Err(e) => {
            format!("[请求失败] {} - 错误: {}", addr, e)
        }
    }
}

// ============================================================
// 第五部分：组合 Future —— JoinAll
// ============================================================

/// 简单的 join_all 实现：并发执行多个 Future
struct JoinAll<F: Future> {
    futures: Vec<Option<Pin<Box<F>>>>,
    results: Vec<Option<F::Output>>,
}

impl<F: Future> JoinAll<F> {
    fn new(futures: Vec<F>) -> Self {
        let len = futures.len();
        JoinAll {
            futures: futures
                .into_iter()
                .map(|f| Some(Box::pin(f)))
                .collect(),
            results: (0..len).map(|_| None).collect(),
        }
    }
}

impl<F: Future> Future for JoinAll<F> {
    type Output = Vec<F::Output>;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Vec<F::Output>> {
        let this = unsafe { self.as_mut().get_unchecked_mut() };
        let mut all_done = true;

        for i in 0..this.futures.len() {
            if this.results[i].is_some() {
                continue; // 已完成
            }

            if let Some(ref mut future) = this.futures[i] {
                match future.as_mut().poll(cx) {
                    Poll::Ready(value) => {
                        this.results[i] = Some(value);
                        this.futures[i] = None; // 释放已完成的 Future
                    }
                    Poll::Pending => {
                        all_done = false;
                    }
                }
            }
        }

        if all_done {
            let results: Vec<F::Output> = this
                .results
                .iter_mut()
                .map(|r| r.take().unwrap())
                .collect();
            Poll::Ready(results)
        } else {
            Poll::Pending
        }
    }
}

// ============================================================
// 第六部分：主函数 —— 使用我们的运行时
// ============================================================

fn main() {
    println!("╔══════════════════════════════════════════╗");
    println!("║    手写 Rust 异步运行时 Demo             ║");
    println!("╚══════════════════════════════════════════╝\n");

    // 创建 Executor 和 Spawner
    let (executor, spawner) = Executor::new();

    // ---- 任务 1: 简单的定时器任务 ----
    spawner.spawn(1, async {
        println!("  [任务1] 开始执行，等待 1 秒...");
        TimerFuture::new(Duration::from_secs(1)).await;
        println!("  [任务1] 1 秒已到！任务完成。");
    });

    // ---- 任务 2: 多个网络请求并发执行 ----
    spawner.spawn(2, async {
        println!("  [任务2] 开始发起多个并发 HTTP 请求...\n");

        // 模拟多个网络请求
        // 注意：这些是真实的 TCP 请求，如果无法连接会超时
        let request1 = HttpFuture::new("httpbin.org", "/get", 80);
        let request2 = HttpFuture::new("httpbin.org", "/ip", 80);
        let request3 = HttpFuture::new("httpbin.org", "/user-agent", 80);

        // 使用我们的 JoinAll 并发等待所有请求
        let results = JoinAll::new(vec![request1, request2, request3]).await;

        println!("\n  [任务2] 所有请求完成！结果：");
        for (i, result) in results.iter().enumerate() {
            println!("  ── 请求 {}: {}", i + 1, result);
        }
    });

    // ---- 任务 3: 模拟多个异步操作的管道 ----
    spawner.spawn(3, async {
        println!("  [任务3] 开始模拟异步流水线...");

        // 步骤 1: 模拟数据获取（等待 500ms）
        println!("  [任务3] 步骤1: 获取数据...");
        TimerFuture::new(Duration::from_millis(500)).await;
        let data = "原始数据-XYZ";
        println!("  [任务3] 步骤1完成: 获取到 '{}'", data);

        // 步骤 2: 模拟数据处理（等待 300ms）
        println!("  [任务3] 步骤2: 处理数据...");
        TimerFuture::new(Duration::from_millis(300)).await;
        let processed = format!("processed({})", data);
        println!("  [任务3] 步骤2完成: '{}'", processed);

        // 步骤 3: 模拟保存结果（等待 200ms）
        println!("  [任务3] 步骤3: 保存结果...");
        TimerFuture::new(Duration::from_millis(200)).await;
        println!("  [任务3] 步骤3完成: 数据已保存！");

        println!("  [任务3] 流水线全部完成! 最终结果: {}", processed);
    });

    // 丢弃 spawner，当所有任务完成后 executor 会退出
    drop(spawner);

    // 运行 Executor
    executor.run();
}
```

### Cargo.toml 依赖

```toml
[package]
name = "mini-async-runtime"
version = "0.1.0"
edition = "2021"

[dependencies]
lazy_static = "1.4"
```

---

## 三、代码架构解析

```
┌─────────────────────────────────────────────────────┐
│                       main()                        │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐       │
│  │  任务 #1   │  │  任务 #2   │  │  任务 #3   │      │
│  │ TimerFuture│  │ HttpFuture│  │  Pipeline  │      │
│  │  (1s 延迟) │  │ (3个请求) │  │  (3步流水) │      │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘       │
│        │              │              │              │
│        └──────────────┼──────────────┘              │
│                       ▼                             │
│              ┌─────────────────┐                    │
│              │    Spawner      │                    │
│              │  (提交到队列)   │                    │
│              └────────┬────────┘                    │
│                       ▼                             │
│              ┌─────────────────┐                    │
│              │   Channel       │  ◄── Waker::wake() │
│              │  (mpsc::channel)│       重新入队      │
│              └────────┬────────┘                    │
│                       ▼                             │
│              ┌─────────────────┐                    │
│              │   Executor      │                    │
│              │  (循环 poll)    │                    │
│              └─────────────────┘                    │
└─────────────────────────────────────────────────────┘
```

### 执行流程详解

```
时间线 ──────────────────────────────────────────────►

Executor:  poll(#1) → Pending    poll(#2) → Pending    poll(#3) → Pending
              │                     │                     │
              ▼                     ▼                     ▼
后台线程:  [线程A: sleep 1s]   [线程B: HTTP req1]    [线程D: sleep 500ms]
                               [线程C: HTTP req2]
                               [线程E: HTTP req3]
              │                     │                     │
              ▼                     ▼                     ▼
           wake(#1)             wake(#2)              wake(#3)
              │                     │                     │
              ▼                     ▼                     ▼
Executor:  poll(#1) → Ready ✅  poll(#2) → ...        poll(#3) → Pending
                                                          │
                                                       (继续步骤2/3...)
```

---

## 四、关键机制深入解读

### 4.1 为什么用 `mpsc::channel` 作为任务队列？

```rust
// 生产者-消费者模型完美匹配 Executor 的需求：
// - Spawner（生产者）可以从任意线程提交任务
// - Executor（消费者）从队列中取出任务执行
// - Waker 也是生产者，将被唤醒的任务重新入队

let (sender, receiver) = mpsc::channel();
```

### 4.2 Waker 的实现原理

Waker 的核心是一个 **虚函数表（VTable）**，这是 Rust 低级抽象的体现：

```rust
// RawWakerVTable 定义了 4 个操作：
RawWakerVTable::new(
    clone,          // 克隆 Waker
    wake,           // 唤醒并消费 Waker
    wake_by_ref,    // 唤醒但不消费 Waker
    drop,           // 销毁 Waker
)

// 这使得 Waker 可以跨越 trait object 的边界，
// 不依赖于具体的 Executor 实现
```

### 4.3 与 tokio 等成熟运行时的对比

| 特性 | 我们的实现 | tokio |
|------|-----------|-------|
| 任务调度 | 单线程，mpsc channel | 多线程 work-stealing |
| I/O 模型 | 后台线程 + 阻塞 I/O | epoll/kqueue/IOCP (非阻塞) |
| 定时器 | 每个定时器一个线程 | 时间轮算法，共享线程 |
| Waker | 手动 VTable | 优化的 waker 实现 |
| 适用场景 | 学习/理解原理 | 生产环境 |

### 4.4 真正的异步 I/O vs 我们的模拟

```
我们的实现（线程模拟异步）：
┌──────────┐     spawn thread      ┌──────────────┐
│  Future   │ ──────────────────>  │ 后台线程       │
│  poll()   │                      │ 阻塞 I/O      │
│  Pending  │ <────── wake() ───── │ 完成后唤醒     │
└──────────┘                       └──────────────┘

真正的异步 I/O（tokio/epoll）：
┌──────────┐    register fd       ┌──────────────┐
│  Future   │ ──────────────────> │ epoll/kqueue  │
│  poll()   │                     │ 内核事件循环   │
│  Pending  │ <──── wake() ────── │ 事件就绪通知   │
└──────────┘                      └──────────────┘
```

---

## 五、不使用 `lazy_static` 的改进版本

如果你不想引入外部依赖，可以使用以下改进方案：

```rust
use std::sync::OnceLock;

static PENDING_TASKS: OnceLock<Mutex<HashMap<usize, Task>>> = OnceLock::new();

fn pending_tasks() -> &'static Mutex<HashMap<usize, Task>> {
    PENDING_TASKS.get_or_init(|| Mutex::new(HashMap::new()))
}
```

或者更优雅的方式——将 pending tasks 存储在 Executor 内部，通过 `Arc` 共享：

```rust
struct Executor {
    ready_queue: Receiver<Task>,
    sender: Sender<Task>,
    pending: Arc<Mutex<HashMap<usize, Task>>>,
}
```

---

## 六、运行效果示例

```
╔══════════════════════════════════════════╗
║    手写 Rust 异步运行时 Demo             ║
╚══════════════════════════════════════════╝

[Spawner] 已提交任务 #1
[Spawner] 已提交任务 #2
[Spawner] 已提交任务 #3
[Executor] 开始运行...

[Executor] 正在 poll 任务 #1...
  [任务1] 开始执行，等待 1 秒...
[Executor] ⏳ 任务 #1 返回 Pending，等待唤醒...

[Executor] 正在 poll 任务 #2...
  [任务2] 开始发起多个并发 HTTP 请求...
  [HTTP] 开始请求: httpbin.org:80/get
  [HTTP] 开始请求: httpbin.org:80/ip
  [HTTP] 开始请求: httpbin.org:80/user-agent
[Executor] ⏳ 任务 #2 返回 Pending，等待唤醒...

[Executor] 正在 poll 任务 #3...
  [任务3] 开始模拟异步流水线...
  [任务3] 步骤1: 获取数据...
[Executor] ⏳ 任务 #3 返回 Pending，等待唤醒...

[Waker] 唤醒任务 #3!
[Executor] 正在 poll 任务 #3...
  [任务3] 步骤1完成: 获取到 '原始数据-XYZ'
  [任务3] 步骤2: 处理数据...
[Executor] ⏳ 任务 #3 返回 Pending，等待唤醒...

[Waker] 唤醒任务 #1!
[Executor] 正在 poll 任务 #1...
  [任务1] 1 秒已到！任务完成。
[Executor] ✅ 任务 #1 已完成!

[Waker] 唤醒任务 #3!
[Executor] 正在 poll 任务 #3...
  [任务3] 步骤2完成: 'processed(原始数据-XYZ)'
  [任务3] 步骤3: 保存结果...
[Executor] ⏳ 任务 #3 返回 Pending，等待唤醒...

...（继续直到所有任务完成）

[Executor] 所有任务已完成，退出。
```

---

## 七、总结

### Rust 异步编程的核心设计哲学

1. **零成本抽象**：async/await 在编译期转换为状态机，无 GC、无运行时开销
2. **运行时可插拔**：语言只定义 `Future` trait，具体运行时由库提供（tokio、async-std、smol 等）
3. **协作式调度**：Future 在 await 点主动让出控制权
4. **类型安全**：Pin 机制在编译期防止自引用问题
5. **惰性求值**：Future 不 poll 就不执行，避免不必要的计算

### 何时使用异步

| 场景                   | 推荐方式                    |
| -------------------- | ----------------------- |
| 大量并发 I/O（Web 服务器、爬虫） | ✅ async/await           |
| CPU 密集计算             | ❌ 使用 `rayon` 或线程池       |
| 少量并发任务               | 🤔 线程可能更简单              |
| 嵌入式/no_std           | ✅ async 很适合（embassy 框架） |