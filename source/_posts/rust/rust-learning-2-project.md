---
title: Rust Learning - Crates and Modules 
date: 2023-03-18 16:25:49
categories:
- rust
tags:
- rust
- learning
---

## RUST 

[Rust 程序设计语言 - Rust 程序设计语言 简体中文版 (kaisery.github.io)](https://kaisery.github.io/trpl-zh-cn/title-page.html)

### Packages

Cargo的一个功能，可以构建、测试和分享crate。包是提供一系列功能的一个或多个crate。一个包会包含一个Cargo.toml文件。Cargo本身也是一个包含构建代码的二进制项目的包。包中可以包含至多一个库crate，和任意多个二进制crate，但是必须至少有一个crate。

一个包目录中

* `src/main.rs`是与包名相同的二进制crate的根crate
* `src/lib.rs`是与包名相同的库crate的根crate
* `src/bin`目录下是这个包中的其他的二进制crate

crate根文件由Cargo传递给rustc来实际构建库或二进制项目。
#### rust版本

##### edition

为了处理rust大版本更新后兼容，在`[package]`中会说明类似`edition = "2021"`版本信息，告诉编译器这个包是对哪个版本兼容的。因此如果项目要使用rust的新特性，需要使用特性对应的版本。基本上3年一个版本，目前最新的为2024.
详细的版本信息和指南在这里[The Rust Edition Guide](https://doc.rust-lang.org/edition-guide/editions/index.html)

使用`cargo fix`可以辅助版本升级。

例如：
* 2015版本兼容rust1.0版本
* 2018版本把async和await作为关键字，所以程序中不能在使用这两个作为变量名

##### rust version

可以为工程指名使用的rust的最低版本，例如在`[package]`下添加`rust-version = "1.91.0"`。如果当前本机安装的rust版本小于指定的版本号，会提示无法编译

```rust
error: rustc 1.88.0 is not supported by the following package:
  memorywalk@0.1.0 requires rustc 1.91.0
```

如果要求版本号小于本地安装的rust版本，cargo会用当前安装的版本编译，不会精确匹配编译器的版本。

* 使用指定的rust版本编译
`cargo +1.91.0 build` 就会用rustup去自动下载1.91.0版本，不过配置的aliyun镜像目录不正确，会下载失败。
* 使用配置文件指定编译版本，在项目根目录下新建`rust-toolchain.toml`，文件中指定rust的版本，下次在`cargo build`时，就会用指定的版本编译
	```toml
	[toolchain]
	channel = "1.91.0" # 也可以写 channel = "stable"
	components = [ "clippy" ]
	```
### Crates

crate是rust在编译时的最小代码单位，可以是一个文件。Crate有两类：库或二进制项目。一般crate都是指的库。

### 依赖

`Cargo.toml`中`[dependencies]`段是当前项目的依赖，cargo在编译时会依次下载依赖库的源代码，并进行编译。如果一个库又依赖其他库，也会先下载被依赖的库，进行编译，从而把整个依赖树下载编译。例如`rand`crate依赖`rand_core v0.9.3`就会下载`rand_core v0.9.3`并进行编译，而不只是下载当前项目直接依赖的crate。

cargo会传递`--extern`选项，告诉rustc在编译时使用的crate，所以当rustc看到代码中的`use rand::Rng;`就直到rand是一个crate，并且也知道去哪里找到这个库文件。

通过`cargo build --verbose`可以查看详细的编译信息
`--extern 'rand=E:\dev\rust\memorywork\target\debug\deps\librand-f6713db433808e1e.rmeta'`

### 项目编译

#### lib项目

cargo使用`--crate-type lib`选项，这样rustc不会去代码中找main函数，同时会生成`.rlib`文件，这时rust的库文件，可以被其他rust程序静态链接使用。

`.rlib`文件中存储了库的类型信息，因此rustc就可以知道程序中使用的crate的features是否在这个crate中。
#### 可执行程序

cargo使用`--crate-type bin`选项，生成一个二进制程序。

`cargo build --release`选项会优化代码，程序执行的更快，但是编译所需的时间更长，不会检查整数溢出，并会跳过`debug_asser!()`断言，生成的调用栈追溯也更不可靠。


### Modules

多个模块构成了一个crate，module用来对一个crate中的代码进行分组，提高可读性和重复使用。模块使用**mod**声明，和python的module类似，也可以看作和c++中的namespace类似。

模块以树结构进行组织，一个模块中的代码默认是私有的，子模块可以访问父模块的成员，但父模块默认不能访问子模块的成员，除非在子模块中将成员声明为**pub**的。同一级的模块之间是可以访问的。

使用super可以访问父一级的内容（方法，结构体，枚举等）。

如果一个模块声明了pub，他的内容对外部来说，还是私有不能访问的，要访问一个模块的内容，必须给具体的内容，例如函数，结构体加上pub。

结构体内的字段默认都是私有，而枚举中的字段都是公开的，不需要给枚举的每个值都增加pub。

`src/lib.rs`文件中

```rust
fn deliver_order() {}
mod front_of_house {
    pub struct Order { // 结构体中的成员默认都是私有的，加上pub外部才能访问
        order_type: String,
        pub order_count: i32,
    }

    impl Order { // 由于Order中有私有成员，所以需要在模块内部提供一个create函数创建Order对象
        pub fn create_order(order_type:&str) ->Order {
            Order { order_type: String::from(order_type), order_count: 1 }
        }
    }

    pub mod hosting {
        pub fn add_to_waitlist() {}
    }

    mod serving {
        fn take_order() {}
    }
    
    fn finish_work() {
        super::deliver_order(); // 访问上一级，即根的接口
    }
}

pub fn eat_at_restaurant() {
    // 绝对路径，crate说明是根
    crate::front_of_house::hosting::add_to_waitlist();
	// 相对路径，这个eat_at_restaurant函数和front_of_house是同一级的。
    front_of_house::hosting::add_to_waitlist();
    // field `order_type` of struct `Order` is private
    //let mut myorder1 = front_of_house::Order {order_count:1, order_type:String::from("food"),}; 
    let mut myorder = front_of_house::Order::create_order("noodles");
    myorder.order_count = 10; // 只能访问pub的成员
}
```

#### use

可以使用use简化模块使用时很长的前缀，和c++的using或python的import类似的作用。use的短路径只能在use所在的特定作用域内使用，如果和use的作用域不同，就不能使用。

```rust
use crate::front_of_house::hosting;

fn eat_at() {
    hosting::add_to_waitlist();
}

mod customer {
    fn eat_at() {
        //failed to resolve: use of undeclared crate or module `hosting`use of undeclared crate or module `hosting`
        hosting::add_to_waitlist();
    }
}
```

use其实也可以直接指定到最后的接口，但是那样以来，使用的地方直接调用接口名字，可能存在不同模块内用相同接口名的情况。所以，一般只是把use指定到模块，类，结构体或枚举。类似python的import，use也有as的语法别名，这样也可以避免冲突。

```rust
use std::fmt::Result;
use std::io::Result as IoResult;
```

使用`pub use`可以把一个名称重导出，相同于这个名字就定义在当前作用域一样。

```rust
pub use crate::front_of_house::hosting;

//在外部使用的地方可以
restarant::hosting::add_to_waitlist(); //跳过了中间的内部的front_of_house
```

use语句可以把多个语句合并简化

```rust
use std::{cmp::Ordering, mem};
use std::io::{self, Write}; // 等价于use std::io和use std::io::Write
use std::collections::*;   // 引用collections下的所有内容
```

#### 模块文件管理

模块文件可以有三种组织方式：

1. 模块使用单独的文件存放，文件名就是模块的名称

不同的模块可以按文件放在其父模块的目录中，编译器根据mod语句定位模块的代码文件的位置。
```rust
└── src
    ├── lib.rs
    ├── main.rs
    └── square.rs
    
// lib.rs，在lib.rs的当前目录中找square.rs或在当前目录下的square目录中找mod.rs，看里面有没有这个模块
pub mod square;

// main.rs
use memorywalk::square::Square;
```
编译器看到了根文件中的square模块声明，就会在根目录中找这个`src/square.rs`文件。

2. 当需要把多个子模块放在一起时，可以使用目录名来创建一个模块，目录中使用`mod.rs`来声明这个模块的子模块

例如有一个模块名称为shape标识形状，它有2个子模块circle和square

```rust
└── src
    ├── lib.rs
    └── main.rs
    └── shape
      ├── circle.rs
      ├── mod.rs
      └── square.rs
      
//lib.rs
pub mod shape;

// square.rs
pub struct Square {
    side: f64,
}  
impl Square {
    pub fn new(side: f64) -> Self {
        Square { side }
    }
    pub fn area(&self) -> f64 {
        self.side * self.side
    }
}

// main.rs
use memorywalk::shape::{Circle, Square};

fn main() {
    let side = 5.0;
    let square = Square::new(side);
    let area = square.area();
    println!("Area of the square with side {} is {}", side, area);
```

3. 使用文件名和目录名相同来创建一个模块

例如在`src/front_of_house.rs`中声明了一个子模块`hosting`，

```rust
└── src
    ├── main.rs
    └── shape.rs
    └── shape
      ├── circle.rs
      └── square.rs
      
// shape.rs 中声明两个子模块，两个子模块的文件放在名字为shape的目录中
pub mod circle;
pub mod square;  

// main.rs中使用
pub mod shape;  // 先声明当前目录下的模块shape
use shape::square::Square;  // 使用shape的子模块  
```

`square`是`shape`的子模块，所以它的模块文件`square.rs`放在他父模块`shape`同名的目录下`src/shape/square.rs`

### IO控制台项目

* 将程序拆成main.rs和lib.rs，程序的逻辑放入lib.rs中
* main中调用lib的run函数

**main.rs**中

```rust
use std::env;
use std::process;
use minigrep::Config;

fn main() {
    let args: Vec<String> = env::args().collect(); // 命令行获取参数转换为string的vec   
	// 当程序返回Result的正常值给config，如果出错使用闭包处理错误信息
    let config = Config::build(&args).unwrap_or_else(|err| {
        eprintln!("args are error: {err}");
        process::exit(1);
    });

    if let Err(e) = minigrep::run(config) { // run 成功并不返回值，所以只关心错误处理
        eprintln!("args are error: {e}");
        process::exit(1);
    }
}
```

**lib.rs**中

```rust
use std::error::Error;
use std::fs;
use std::env;

pub fn run(config: Config) -> Result<(), Box<dyn Error>>{ // Error trait, dyn表示无需指定具体返回值类型
    let contents = fs::read_to_string(config.file_path)?;
    // ？会从函数中返回错误值并让调用者处理
    
    let results = if config.ignore_case {
        search_case_insensitive(&config.query, &contents)
    } else {
        search(&config.query, &contents)
    };
    for line in results {
        println!("{line}")
    }
    Ok(()) // 没有具体地内容要返回，那就返回unit()
}

pub struct Config {
    pub query: String,
    pub file_path: String,
    pub ignore_case: bool,
}

impl Config {
    pub fn build(args: &[String]) ->Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("Not enough args");
        }

        let query = args[1].clone();
        let file_path = args[2].clone();
        // 获取环境变量中IGNORE_CASE是否设置，但不关心他的值是什么
        let ignore_case = env::var("IGNORE_CASE").is_ok();
    
        Ok(Config {query, file_path, ignore_case})
    }
}
// 返回值的生命周期和输入的被查询内容的生命周期应该一样
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str>  {
    let mut results = Vec::new();
    for line in contents.lines() {
        if line.contains(query) {
            results.push(line);
        }
    }
    results
}

pub fn search_case_insensitive<'a>(query: &str, contents: &'a str) -> Vec<&'a str>  {
    let mut results = Vec::new();
    let query = query.to_lowercase();
    for line in contents.lines() {
        if line.to_lowercase().contains(&query) {
            results.push(line);
        }
    }
    results
}
// 单元测试
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn case_sensitive() {
        let query = "day";
        let contents = "\
best and
colorful days";

        assert_eq!(vec!["colorful days"], search(query, contents));
    }

    #[test]
    fn case_insensitive() {
        let query = "Day";
        let contents = "\
best and
colorful days";

        assert_eq!(vec!["colorful days"], search_case_insensitive(query, contents));
    }
}
```

* cargo test执行其中的单元测试用例

* windows中设置环境变量，并运行程序

`PS E:\code\rust\minigrep> $Env:IGNORE_CASE=1; cargo run Body poem.txt`

* 使用`eprintln!`将错误信息输出到标准错误流，将正常输出到文件中。

`cargo run BOdy  poem.txt > output.txt`