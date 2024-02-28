---
title: Rust Learning-Test
date: 2024-02-25 23:15:49
categories:
- programming
tags:
- rust
- learning
---

## RUST Test 

[Rust 程序设计语言 - Rust 程序设计语言 简体中文版 (kaisery.github.io)](https://kaisery.github.io/trpl-zh-cn/title-page.html)

### Test Function

一个测试函数执行三个任务：

1. 初始设置测试的数据和状态
2. 执行需要测试的代码
3. 判断代码执行结果是否与预期一致

定义一个测试函数时，需要在这个函数前用`#[test]`注解，这样`cargo test`执行时，就会运行这些测试函数，并汇报最终通过与否的结果。

#### 简单测试例子

当创建一个rust的lib库工程时，一个测试模块会自动生成。

执行`cargo new plus --lib`创建一个名称为plus的lib库。

默认生成的`lib.rs`代码如下

```rust
pub fn add(left: usize, right: usize) -> usize {
    left + right
}

#[cfg(test)]
mod tests {
    use super::*; // 测试模块可以使用外部的所有接口，用来测试

    #[test]
    fn it_works() {
        let result = add(2, 2);
        assert_eq!(result, 4);
    }
    #[test]
    fn it_not_work() {
        let result = add(2, 1);
        assert_eq!(result, 4);
    }
}

```

与普通执行程序不同，这里执行`cargo test`就会执行我们发的测试.

```shell
running 2 tests
test tests::it_works ... ok
test tests::it_not_work ... FAILED

failures:

---- tests::it_not_work stdout ----
thread 'tests::it_not_work' panicked at src\lib.rs:18:9:
assertion `left == right` failed
  left: 3
 right: 4
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

failures:
    tests::it_not_work

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

输出结果说明有个一个测试被执行，结果为ok。总的测试结果也是Ok。通过`cargo test`指定具体函数名字，可以控制执行匹配字串的测试用例，也可以控制过滤不执行哪些测试用例。measure用来性能测试，目前只在每日编译版本中支持。

rust可以编译程序api文档中的代码，`Doc-tests`就是文档中的代码执行测试用例

#### 使用断言assert!宏

当`assert!`宏中的值为false时，会调用`panic!`宏触发测试执行失败

`assert!`用来简单判断一个值是否是true

`assert_eq!` 用来判断两个值是否相等，当不相等时，会打印出来两个值。 `assert_ne!`用来判断两个值不相等。这两个宏使用传入参数的`debug`格式化输出和使用`==`和`!=`进行比较，对于自定义的结构体或枚举，需要实现 `PartialEq` 和`Debug` traits。由于这两个trait都是derivable 可获得的（编译器可以自动生成默认实现代码），所以可以在自定义的结构体前加上 `#[derive(PartialEq, Debug)]`注解，就可以获得trait的默认实现。

#### 添加自定义的失败信息

在`assert!`、`assert_eq!`、 `assert_ne!`的比较结果的参数后还可以增加一一个 `format!` 宏格式化的字串来输出失败信息。

```rust
    #[test]
    fn it_not_work() {
        let result = add(2, 1);
        assert_eq!(result, 4, "failed with result = {}", result);
    }
}
// assertion `left == right` failed: failed with result = 3
```

程序在执行失败时，附带其中的错误信息。

#### 检查被测函数输出panic

除了检查被测函数有正确输出值，我们还要检查函数是否有正确处理错误异常，如果一个被测函数输出了panic，那么这个测试就通过。这时可以在测试函数上增加`#[should_panic]`属性。并且还可以指定我们预期panic中输出的字串有一定有哪些信息。

```rust
pub fn add(left: usize, right: usize) -> usize {
    if left > 100 {
        panic!("left too large with value {}", left)
    } else if right > 100 {
        panic!("right too large with value {}", right)
    }
    left + right
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic(expected ="right too large")]
    fn it_panic() {
        let result = add(150, 50);
        assert_eq!(result, 200, "failed with result = {}", result);
    }
}
```

最终会输出函数panic输出的信息中没有预期的字串

```powershell
thread 'tests::it_panic' panicked at src\lib.rs:3:9:
left too large with value 150
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
note: panic did not contain expected string
      panic message: `"left too large with value 150"`,
 expected substring: `"right too large"`
```

#### 使用 `Result<T, E>` 作为返回值

测试函数还可以使用 `Result<T, E>` 作为返回值，当测试通过时返回Ok，失败时返回Err。使用 `Result<T, E>` 作为测试函数的返回值时，不能再使用`#[should_panic]`属性。

```rust
pub fn add(left: usize, right: usize) -> usize {
    left + right + 1
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() -> Result<(), String>{
        let result = add(2, 2);
        if result == 4 {
            Ok(()) // pass
        } else {
            Err(String::from("result should be 4")) // failed
        }
    }
}
---- tests::it_works stdout ----
Error: "result should be 4"
```

### Test run

`cargo test --`后面的选项是给`cargot test`使用的，例如`cargo test --hlep`是列出`cargo test`的帮助信息

#### 测试用例顺序执行

当执行多个测试时，默认这些测试是并发执行的，这样执行的更快。使用`cargo test -- --test-threads=1`所有的测试都在一个线程中执行，不会因为并发导致互相影响结果

#### 测试函数输出

当测试pass时，在测试函数以及被测函数中的`println!()`都不会输出到标准输出，只有测试失败才会输出。

`cargo test -- --show-output`可以在测试pass的时候，还能输出函数中的`println!()`

#### 执行指定的测试函数

`cargo test 测试函数名称`例如`cargo test it_not_work`就只执行`it_not_work`这个测试函数，其他的测试函数不执行。

`cargo test 测试名称匹配字串`可以过滤执行多个测试函数，例如`cargo test work`表示执行所有名称中有`work`字串的测试函数。

#### 忽略测试函数

在测试函数名称前加上`#[ignore]`，就可以在默认执行`cargo test`把它忽略不执行，这对于非常耗时的测试用例非常有用。

使用`cargo test -- --ignored`来只执行标注了ignore的测试函数。

使用`cargo test -- --include-ignored`可以执行所有的测试函数。

```rust
#[test]
#[ignore]
fn long_time_work() {
  assert_eq!(1, 1);
}
```

默认`cargo test`执行时，会提示哪些函数被忽略了。

```shell
running 3 tests
test tests::long_time_work ... ignored
test tests::it_not_work ... ok
test tests::it_works ... ok
```

### Test Organization

单元测试用来测试每一个模块内部的接口包括私有的接口

集成测试是像外部应用使用库一样测试这个库的外部接口，它只测试公共接口，且同时可能测试多个模块。

#### 单元测试

单元测试的测试代码可以和被测的模块代码在同一个文件中。通过在测试模块前加`#[cfg(test)]`，告诉编译器只有执行`cargo test`的时候才会编译这个测试模块，这样发布的程序中就不会包含测试的代码。

测试私有函数时对于C++应该很难实现，对于rust虽然测试模块是一个独立的作用域，通过测试模块中使用`use super::*`，这样测试模块里面就可以使用它所在的父模块的所有成员。

```rust
pub fn add_two(a: i32) -> i32 {
    internal_adder(a, 2)
}
// 没有pub的私有模块函数
fn internal_adder(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*; // 可以访问这个test模块的父模块的所有函数

    #[test]
    fn internal() {
        assert_eq!(4, internal_adder(2, 2));
    }
}
```

#### 集成测试

集成测试针库整体测试。

##### 集成测试目录结构

和`src`文件同级创建一个`tests`目录，cargo会把这个`tests`目录中的每一个`rs`文件作为一个独立的crate。这个目录中的文件只有在执行`cargo test`时候才会被编译执行。

```rust
plus
├── Cargo.lock
├── Cargo.toml
├── src
│   └── lib.rs
└── tests
    └── integration_test.rs
```

integration_test.rs中的内容如下，需要引用一下被测试的库。由于rust会自动把tests目录下的文件作为测试代码，所以不需要增加`#[cfg(test)]`和测试模块，每一个文件都是一个独立的测试模块了。

```rust
use plus;

#[test]
fn test_add() {
    let result = plus::add(2, 2);
    println!("The result is {}", result);
    assert_eq!(result, 4);
}
```

执行`cargo test`后，会先执行库代码中的单元测试，再执行外层的集成测试。如果单元测试有用例执行失败，就不会执行外部的集成测试。

`cargo test --test integration_test`表示只执行文件名称为`integration_test`中的测试用例，库源代码中的单元测试也不会被执行。

如果工程只是一个二进制程序类型，且只有`main.rs`，而没有`lib.rs`，那么就不能使用`tests`目录来创建集成测试，因为只有lib库类型的代码才会暴露模块接口给外部使用，而应用程序不会。一般一个项目会把逻辑和算法放在lib中，main中只是调用库的接口。

##### 集成测试目录中使用公共子模块

一些多个测试模块都要使用的公共方法可以放在`tests/common/mod.rs`文件中，这样编译器不会把mod.rs中的函数作为测试函数执行。

```rust
├── Cargo.lock
├── Cargo.toml
├── src
│   └── lib.rs
└── tests
    ├── common
    │   └── mod.rs
    └── integration_test.rs
```

例如tests/common/mod.rs中定义了一个公共准备测试的函数

```rust
pub fn setup() {
    println!("prepare for the test");
}
```
在测试文件中就可以使用common这个模块
```rust
use plus;

mod common;

#[test]
fn test_add() {
    common::setup();
    let result = plus::add(2, 2);
    println!("The result is {}", result);
    assert_eq!(result, 4);
}
```

使用`cargo test --test integration_test -- --show-output`只执行这个集成测试文件，并把测试函数中的输出也打印出来。第一个`--test`是给`cargo test`的参数，后面的参数相当于是给这个测试程序的参数。 
