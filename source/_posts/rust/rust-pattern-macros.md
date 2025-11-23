---
title: Programming Rust - Macros
date: 2025-10-18 15:42:49
categories:
  - programming
tags:
  - rust
  - macro
---

# RUST Macros

## 宏

宏是一种为写其他代码而写代码的方式。

宏在程序代码编译为机器码之前会被展开为rust代码，所以它与函数调用不同，宏必须在使用前定义。rust中的宏和c++中的宏类似，但是rust的宏有语法检查，不像C++的宏只是纯粹的文本展开。

```rust
// 一个断言宏
assert_eq!(gcd(6, 10), 2);
// 上面断言宏展开
match (&gcd(6, 10), &2) {
    (left_val, right_val) => {
        if !(*left_val == *right_val) {
            panic!("assertion failed: `(left == right)`, \
                (left: `{:?}`, right: `{:?}`)", left_val, right_val);
        }
    }
}
```

宏在使用时使用exclamation point**感叹号**作为标记
### 声明宏

详细教程[“The Little Book of Rust Macros”](https://veykril.github.io/tlborm/)

对传给宏的源代码字面值与模式匹配，如果匹配成功，模式的代码会替换为传递给宏的代码，最终替换到模板代码中。声明宏可以使用`macro_rules!`来定义，定义的格式一般为
```rust
( pattern1 ) => ( template1 );
( pattern2 ) => ( template2 );
```
即把一个模式替换为一个模板中的内容，其中的`()`也可以用`[]`或`{}`，对rust而言这三个符号没有区别。因此使用一个宏的时候，这三种符号都可以使用，只是`{}`不需要额外的`;`作为语句结束。通常情况下,`assert_eq!`使用`()`，`vec!`使用`[]`，`macro_rules!`使用`{}`

宏定义的[模式语法](https://doc.rust-lang.org/reference/macros-by-example.html)和普通的rust模式匹配的语法不同，宏定义的模式匹配的是代码结构，普通的模式匹配的是值。
#### 宏展开

`assert_eq`的定义如下，定义宏时，名字后面不需要`!`，这里的`($left:expr, $right:expr $(,)?)`部分就是模式，其中expr标识匹配一个表达式。在模板中使用`$left`，不能带类型`expr`。
**注意**：这里把模式变量`$left`转换为本地变量`left_val`在模板中使用，因为如果直接使用原始的表达式，rust会简单的把这个表达式替换在模板中，如果这个表达式是`letter.pop()`这种每次执行都会产生变化的，在模板中调用多次，值已经不是预期的调用一次的值了，所以使用match把表达式只计算一次，并把值保存重复使用。至于为什么用match，而不用let，没有特别的原因，也可以用let。另外这里使用了`&$left`引用，是为了避免把宏参数的所有权移入的宏内部，导致外部无法再使用参数，例如参数不是这里的整数，而是String类型，就会把变量move到宏内部，宏后面的代码如果想继续使用这变量就会无法访问了。

`#[macro_export]`注解说明导入这个宏所在的crate，就可以使用这个宏，否则不能引用这个宏

宏定义中，使用`$`作为变量前缀，说明这个变量是一个宏变量

```rust
#[macro_export]
#[stable(feature = "rust1", since = "1.0.0")]
#[rustc_diagnostic_item = "assert_eq_macro"]
#[allow_internal_unstable(panic_internals)]
macro_rules! assert_eq {
    ($left:expr, $right:expr $(,)?) => {
        match (&$left, &$right) {
            (left_val, right_val) => {
                if !(*left_val == *right_val) {
                    let kind = $crate::panicking::AssertKind::Eq;
                    // The reborrows below are intentional. Without them, the stack slot for the
                    // borrow is initialized even before the values are compared, leading to a
                    // noticeable slow down.
                    $crate::panicking::assert_failed(kind, &*left_val, &*right_val, $crate::option::Option::None);
                }
            }
        }
    };
    ($left:expr, $right:expr, $($arg:tt)+) => {
        match (&$left, &$right) {
            (left_val, right_val) => {
                if !(*left_val == *right_val) {
                    let kind = $crate::panicking::AssertKind::Eq;
                    // The reborrows below are intentional. Without them, the stack slot for the
                    // borrow is initialized even before the values are compared, leading to a
                    // noticeable slow down.
                    $crate::panicking::assert_failed(kind, &*left_val, &*right_val, $crate::option::Option::Some($crate::format_args!($($arg)+)));
                }
            }
        }
    };
}
```

对于C++，`#define ADD_ONE(n) n + 1` 这样的宏，如果这样使用`ADD_ONE(1) * 10`或`ADD_ONE(1 << 4)`都会产生非预期的结果，但是rust的宏会在把一个表达式复制的时候自动加上括号。

#### 宏重复

vec!的实现框架如下，这个宏定义了三个规则，编译器拿到代码`vec![1, 2, 3]`后会按顺序逐个规则进行匹配，找到第一个有效匹配。

```rust
macro_rules! vec {
    ($elem:expr ; $n:expr) => {// vec![0, 100]
        ::std::vec::from_elem($elem, $n)
    };
    ( $( $x:expr ),* ) => { // vec![1, 2, 3]
        <[_]>::into_vec(Box::new([ $( $x ),* ]))
    };
    ( $( $x:expr ),+ ,) => {// 匹配列表末尾是逗号的情况
        vec![ $( $x ),* ]
    };
}
// 还可以使用push执行多次的方法实现，对于第二个规则
( $( $x:expr ),* ) => {
    {
        let mut v = Vec::new();
        $( v.push($x); )* // 对于表达式列表$x的每一个表达式都执行一次v.push()，最后的*表示重复多次
        v
    }
};
```

其中第二个规则的模式`$( PATTERN ),`表示使用`,`分隔，重复`PATTERN`多次，后面的`*`表示重复0或多次，和正则表达式一样，可以使用`+`表示重复1或多次，`?`表示0或1次。`$x:expr`在这里不是一个表达式，而是一个表达式列表。
`<[_]>`表示某种类型的切片，这个类型由rust自己推导出来。
注意：`fn()`, `&str`, or `[_]`这种特殊字符的表达式需要使用`<>`括起来

#### 内建宏

一部分宏在rustc编译器内部实现，而不是通过`macro_rules!`来定义。

* `file!()` 当前文件名的字串值
* `line!()`当前行号
* `stringify!(...tokens...)`把rust代码元素以字串值显示出来，如果参数是宏，这个宏不会被展开。`stringify!(line!())`只会输出“line!()”。
* `concat!(str0, str1, ...)`把列表中的字串拼接为一个字串
* `cfg!(...)`获取当前编译配置是否为括号中值的boolean值。`cfg!(debug_assertions);`debug模式下返回值为true。
* `env!("VAR_NAME")`获取指定的环境变量的字串值，例如`env!("CARGO_PKG_VERSION");`得到字串`0.1.0`
* `option_env!("VAR_NAME")`同上，只是返回一个option，如果环境变量不存在返回None
* `include!("file.rs")`把另一个rust代码文件扩展进来
* `include_str!("file.txt")`把一个文本文件读入到一个`&'static str`中，`const COMPOSITOR_SHADER: &str = include_str!("../resources/compositor.glsl");
* `include_bytes!("file.dat")`把一个二进制文件读入到`&'static [u8]`中
* `matches!(value, pattern)` 相当于以下代码，当一个value匹配了pattern，返回true
    ```rust
    match value {
        pattern => true,
        _ => false
    }
    ```
* `unimplemented!()`如果代码执行到这里会`panic`，`todo!()`表示这段代码还需要实现`not yet implemented: `

#### 宏调试

使用`cargo-expand`查看展开后的代码，安装`cargo install cargo-expand` 后，项目目录下执行`cargo expand`就可以查看展开后的代码。

例如函数
```rust
fn test_macros() {
    let data = vec![1, 2, 3];
    println!("data is {:?}", data);
}
```
对应的输出为
```rust
fn test_macros() {
    let data = <[_]>::into_vec(::alloc::boxed::box_new([1, 2, 3]));
    {
        ::std::io::_print(format_args!("data is {0:?}\n", data));
    };
}
```

使用`trace_macros!(true)`让rustc输出宏的名称和参数，只有这个宏有效区间的宏展开会输出

```rust
#![feature(trace_macros)]

fn test_macros() {
    trace_macros!(true);
    let data = vec![1, 2, 3];
    trace_macros!(false); // 这个代码之后宏展开不会输出
    println!("data is {:?}", data);
}
```
输出
```rust
note: trace_macro
  --> src\bin\lang.rs:90:16
   |
90 |     let data = vec![1, 2, 3];
   |                ^^^^^^^^^^^^^
   |
   = note: expanding `vec! { 1, 2, 3 }`
   = note: to `< [_] > :: into_vec($crate :: boxed :: box_new([1, 2, 3]))`
```

### 过程宏(Procedural macros)

过程宏像函数一样接收rust代码作为输入，在这些代码上进行操作，然后输出另一些代码

过程宏需要定义在特殊类型的crate中

定义过程宏的函数接收一个`TokenStream` 作为输入并生成 `TokenStream` 作为输出。函数上还有一个属性指明了创建的过程宏的类型。在同一 crate 中可以有多种过程宏。

`TokenStream` 是`proc_macro` crate 里定义的代表一系列 token 的类型。宏所处理的源代码组成了输入 `TokenStream`，宏生成的代码是输出 `TokenStream`。
#### 派生宏

派生宏可以为注解的代码额外添加功能的代码，例如为一个struct生成trait的方法实现。例如`#[derive(Debug)]`。

##### 创建过程宏

假设有一个库名称为breakingbad，它有一个trait叫SayMyName，现在要为这个trait定义过程宏`breakingbad_derive`，方便所有实现这个trait的结构都可以SayMyName。

1. 使用`cargo new breakingbad --lib`创建一个库crate
2. 在lib.rs中定义这个库的trait和它的方法
```rust
pub trait SayMyName {
    fn say_macro();
}
```
3. 按命名习惯创建库的过程宏的crate名字为`libname_derive`，这里在库的目录下直接`cargo new breakingbad_derive --lib`创建派生过程宏的工程
4. 修改过程宏工程toml文件，配置lib为过程宏，并添加syn和quote的依赖。`syn` crate 将Rust 代码字符串解析成为一个可以操作的数据结构。`quote` crate 则将 `syn` 解析的数据结构转换回 Rust 代码。
```toml
[lib]
proc-macro = true  

[dependencies]
syn = "2.0"
quote = "1.0"
```
5. 在过程宏的lib.rs文件中定义一个过程宏，一般都分两步实现，先用syn的parse解析代码字串为结构，再根据结构的信息生成代码字串。
```rust
use proc_macro::TokenStream;
use quote::quote;

#[proc_macro_derive(SayMyName)]
pub fn breakingbad_derive(input: TokenStream) -> TokenStream {
    // 使用syn将输入的Rust 代码TokenStream构建成我们可以操作的语法树 DeriveInput类型
    let ast = syn::parse(input).unwrap();

    // 生成 trait 的实现。
    impl_say_macro(&ast)
}

fn impl_say_macro(ast: &syn::DeriveInput) -> TokenStream {
    let name = &ast.ident; // identity 是类型名字
    let generated = quote! {// quote! 宏返回需要的代码
        impl SayMyName for #name {
            fn say() {
                // stringify!(#name) 把输入的表达式转换为硬编码字符串，而不是计算表达式的值，节省一次内存分配
                println!("My name is {}!", stringify!(#name));
            }
        }
    };
    generated.into() // 转换为TokenStream
}
```
一个DeriveInput结构体内容类似如下
```rust
DeriveInput { 
    // --snip-- 
    ident: Ident { 
        ident: "Heisenberg", 
        span: #0 bytes(95..103) 
    }, 
    data: Struct( DataStruct { 
        struct_token: Struct, fields: Unit, semi_token: Some( Semi ) 
    } ) 
}
```
5. 在项目toml文件中`[dependencies]`段下添加过程宏crate的依赖`breakingbad_derive = { path = "breakingbad_derive" }`，项目目录新建example测试程序 `\breakingbad\examples\derive_example.rs`
```rust
use breakingbad::SayMyName;
use breakingbad_derive::SayMyName;

#[derive(SayMyName)]
struct Heisenberg;

fn main() {
    // The generated impl will print the type name.
    Heisenberg::say();
}
```
6. 执行`cargo run --example derive_example -q`来运行example程序，输出`My name is Heisenberg!`

#### 类属性宏(Attribute-Like)

派生宏只能为derive属性生成代码，只能用于结构体和枚举；属性宏可以创建新的属性，它可以应用于其他类型，如函数上。

例如web框架一般提供的`#[route(GET, "/")]`就是框架库定义的属性名称为route的过程宏。这个过程宏的定义一般如下：

```rust
#[proc_macro_attribute]
pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream {
```
第一个参数attr是属性的内容，即例子中的`GET, "/"`，第二个参数为注解的函数。属性宏的定义方法和派生宏一样。
#### 类函数宏(Function-like)

函数宏的定义像函数的调用，它可以接收任意数量的参数。和另外两种过程宏一样，它也接收一个`TokenStream` 参数，它定义的函数处理这个输入参数，并输出`TokenStream`。

例如`sql!`宏用来检查输入的sql语句是否合法，而不是简单的像`macro_rules!`那样替换代码。它的定义如下
```rust
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {
}
```

使用时和函数调用类似
```rust
let sql = sql!(SELECT * FROM posts WHERE id=1);
```
