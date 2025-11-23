---
title: Programming Rust - Macros
date: 2025-10-18 15:42:49
categories:
  - programming
tags:
  - rust
---

## RUST Macros

### 宏

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

对代码模板进行简单替换。声明宏可以使用`macro_rules!`来声明，定义的格式一般为
```rust
( pattern1 ) => ( template1 );
( pattern2 ) => ( template2 );
```
即把一个模式替换为一个模板中的内容，其中的`()`也可以用`[]`或`{}`，对rust而言这三个符号没有区别。因此使用一个宏的时候，这三种符号都可以使用，只是`{}`不需要额外的`;`作为语句结束。通常情况下,`assert_eq!`使用`()`，`vec!`使用`[]`，`macro_rules!`使用`{}`

`assert_eq`的定义如下，定义宏时，名字后面不需要`!`

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

### 过程宏

#### 函数宏
#### 派生宏

例如`#[derive(Debug)]`，会额外添加功能的代码，为下面的结构体增加实现代码。


#### 属性宏

