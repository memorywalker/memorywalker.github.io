---
title: Rust WebAssembly
date: 2026-01-25T10:18:00
categories:
  - rust
tags:
  - rust
---

## Rust WebAssembly

https://rustwasm.github.io/docs/book/
https://wasm.rust-lang.net.cn/docs/book/

Rust and WebAssembly Working Group因为活跃度太低，已经被归档了。
https://blog.rust-lang.org/inside-rust/2025/07/21/sunsetting-the-rustwasm-github-org/

WebAssembly (wasm) 是一种简单的机器模型和可执行格式，具有 [广泛的规范](https://github.webassembly.net.cn/spec/)。它旨在可移植、紧凑，并在接近原生速度的情况下执行。

wasm 并没有对它的宿主环境做出任何假设。目前wasm 主要与 JavaScript相关（包括 Web 上和 [Node.js](https://node.org.cn/) 上的）。

WebAssembly 包含两种格式：

1. `.wat` 文本格式（称为 `wat`，代表 "**W**eb**A**ssembly **T**ext"）使用 [S 表达式](https://en.wikipedia.org/wiki/S-expression)，与 Scheme 和 Clojure 等 Lisp 语言家族有些相似。    
2. `.wasm` 二进制格式是更低级的，旨在直接供 wasm 虚拟机使用。它在概念上类似于 ELF 和 Mach-O。

WebAssembly 具有非常简单的 [内存模型](https://github.webassembly.net.cn/spec/core/syntax/modules.html#syntax-mem)。wasm 模块可以访问单个“线性内存”，它本质上是一个字节的扁平数组。这个 [内存可以按页面大小（64K）的倍数增长](https://github.webassembly.net.cn/spec/core/syntax/instructions.html#syntax-instr-memory)。它不能缩小。

### 基本教程

#### 安装依赖

 1. 安装目标 `rustup target add wasm32-unknown-unknown`

2. `wasm-pack` 是一个构建、测试和发布 Rust 生成的 WebAssembly的工具

3. `wasm-bindgen`是一个库，通过 `#[wasm_bindgen]` 宏来在rust代码中定义哪些rust的接口提供给js调用，rust中又可以调用哪些js的接口。

4. 安装`cargo install wasm-bindgen-cli`，在构建工程时`wasm-pack build`需要调用`wasm-bindgen-cli`
5. 安装npm


#### rust工程

1. 新建一个rust lib工程`cargo new --lib wasm-demo`
2. 修改`cargo.toml`文件
```toml
[lib]
crate-type = ["cdylib", "rlib"] # 重要：声明将编译为动态库（.wasm）  

[dependencies]
wasm-bindgen = "0.2.84"  

[profile.release]
# Tell `rustc` to optimize for small code size.
opt-level = "s"
```

3. lib.rs中测试代码如下
```rust
use wasm_bindgen::prelude::*; 
#[wasm_bindgen]
extern "C" {
    fn alert(s: &str); // rust中可以调用的接口
} 
#[wasm_bindgen]
pub fn greet() { // rust对外提供的接口
    alert("Hello, Wasm-Demo!");
}
```

4. `wasm-pack build`编译rust工程后，会在当前工程目录下生成pkg目录，其中有`wasm_demo_bg.wasm`和`wasm_demo.js`，后者可以给web工程中的js代码调用的接口.

```
[INFO]: Checking for the Wasm target...
[INFO]: Compiling to Wasm...
   Compiling wasm-demo v0.1.0 (E:\dev\rust\wasm-demo)
    Finished `release` profile [optimized] target(s) in 0.18s
[INFO]: Optimizing wasm binaries with `wasm-opt`...
[INFO]: Optional fields missing from Cargo.toml: 'description', 'repository', and 'license'. These are not necessary, but recommended
[INFO]: :-) Done in 33.24s
[INFO]: :-) Your wasm pkg is ready to publish at E:\dev\rust\wasm-demo\pkg.
```

#### Web工程

1. 在工程目录下执行`npm init wasm-app www` 会在当前目录下，新建一个www目录，并在其中从github下载`https://github.com/rustwasm/create-wasm-app`提供的模板工程
2. 由于模板工程还是7年前的版本，最新的npm直接安装依赖后运行不起来，需要以下修改：
    1. 修改`package.json`中的依赖为新版本webpack，并添加一个新的依赖为当前工程编译出来的pkg
    
        ```json
        "dependencies": {                     
            "wasm-demo": "file:../pkg"
        },
        "devDependencies": {
            "webpack": "^5.104.1",
            "webpack-cli": "^6.0.1",
            "webpack-dev-server": "^5.2.3",
            "copy-webpack-plugin": "^13.0.1"
        }
        ```

    2. 修改webpack的配置文件`webpack.config.js`

        ```json
        const CopyWebpackPlugin = require("copy-webpack-plugin");
        const path = require('path');        
        module.exports = {
          entry: "./bootstrap.js",
          output: {
            path: path.resolve(__dirname, "dist"),
            filename: "bootstrap.js",
          },
          mode: "development",
        
          experiments: {
            asyncWebAssembly: true, // 启用异步加载 WASM
          },
        
          plugins: [
            new CopyWebpackPlugin({ patterns: [{ from: 'index.html' }] })
          ],
        };
        ```

3. 进入www目录中安装依赖`npm install`
4. 运行web工程`npm run start`

    ```
    E:\dev\rust\wasm-demo\www>npm run start
    create-wasm-app@0.1.0 start
    webpack-dev-server    
    <i> [webpack-dev-server] Project is running at:
    <i> [webpack-dev-server] Loopback: http://localhost:8080/, http://[::1]:8080/
    <i> [webpack-dev-server] On Your Network (IPv4): http://192.168.1.14:8080/
    ```

5. 打开浏览器http://localhost:8080/ 可以看到弹出的提示信息


[完成Demo工程](/uploads/rust/wasm-demo.7z)
