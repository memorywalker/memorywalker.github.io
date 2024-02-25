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
    use super::*;

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

#### 使用断言

### Test run

### Test Organization