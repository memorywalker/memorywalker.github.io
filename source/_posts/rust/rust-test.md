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

### Test Organization