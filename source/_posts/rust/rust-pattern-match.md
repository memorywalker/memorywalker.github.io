---
title: Rust Learning-Patterns and Matching
date: 2024-02-18 10:42:49
categories:
- programming
tags:
- rust
- learning
---

## RUST Patterns and Matching

[Rust 程序设计语言 - Rust 程序设计语言 简体中文版 (kaisery.github.io)](https://kaisery.github.io/trpl-zh-cn/title-page.html)

### 模式

Pattern是一种语法，用来匹配类型中的结构，和match配合使用。模式由以下几种类型组成：

1. Literals 字面值，写死的字串或数字
2. 结构的数组，枚举，结构体或元组
3. 变量
4. 通配符
5. 占位符

#### 模式使用场景

##### match分支

```rust
match VALUE {
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
}
```

match表达式所有可能值都必须被处理。一种确保处理所有情况的方法是在最后一个分支使用可以匹配所有情况的模式，如使用`_`模式匹配所有情况。

##### if let条件

if let用来处理简单匹配一种情况的场景，当然也可以使用else来处理其他情况。if let, else if, else if let的条件可以是不相关的。编译器不会对if let的所有情况是否都覆盖了进行检查。if let可以和match一样使用覆盖变量 shadowed variables ，例如 `if let Ok(age) = age` 引入了一个新的shadowed `age` 变量，它包含了Ok变量中的值，它的作用域从if let的大括号的范围开始，所以`age > 30`中的age只能在if let代码块的内部有效。

```rust
fn main() {
    let age: Result<u8, _> = "34".parse();
    if let Ok(age) = age {
        if age > 30 {
            println!("Using purple as the background color");
        } else {
            println!("Using orange as the background color");
        }
    }
}
```

##### while let条件

只要while let后面的模式始终匹配，循环就一直执行。下面例子中只有pop返回了None的时候才会结束循环

```rust
let mut stack = Vec::new();

stack.push(1);
stack.push(2);
stack.push(3);

while let Some(top) = stack.pop() {
    println!("{}", top);
}
```

##### for循环

for之后的值就是pattern，例如`for x in y`中,x就是一个模式。 `enumerate`  方法返回值和索引，一起放在一个元组中，例如第一次执行返回  `(0, 'a')`，所以可以使用  `(index, value)` 来解构元组中的元素。

```rust
let v = vec!['a', 'b', 'c'];

for (index, value) in v.iter().enumerate() {
    println!("{} is at index {}", value, index);
}
a is at index 0
b is at index 1
c is at index 2
```

##### let语句

```rust
let PATTERN = EXPRESSION;
```

例如` let x = 5 `中x就是一种模式，它表示把所有匹配到的值绑定到变量x的模式。下面的元组匹配更直观的提现了模式匹配，三个数字分别匹配到对应的xyz.

```rust
let (x, y, z) = (1, 2, 3);
let (x, y) = (1, 2, 3); // error
```

##### 函数参数

函数参数和let语句类似，形参变量就是模式，下面的实参 `&(3, 5)` 匹配模式 `&(x, y)` 

```rust
fn print_coordinates(&(x, y): &(i32, i32)) {
    println!("Current location: ({}, {})", x, y);
}

fn main() {
    let point = (3, 5);
    print_coordinates(&point);
}
```

### 模式匹配的可反驳性

模式有两种形式 **refutable**可反驳的和**irrefutable**不可反驳的 。

不会出现匹配失败，可以匹配所有可能值的模式为不可反驳的，，例如` let x = 5 `中x可以匹配所有值不会匹配失败

可能匹配失败的模式为可反驳的，例如 `if let Some(x) = a_value` ，如果值为None，Some(x)模式就会匹配失败。

函数参数、let语句、for循环只能接受不可反驳的模式，因为他们不能处理模式匹配失败的情况。对于if let、while let表达式可以接受不可反驳模式和可反驳模式，但是对于不可反驳模式由于模式不会失败，没有实际意义，所以编译器会提示编译警告。

### 模式语法



