---
title: Rust Learning-Smart Pointers
date: 2024-01-14 15:42:49
categories:
- programming
tags:
- rust
- learning
---

## RUST Smart Pointers 

[Rust 程序设计语言 - Rust 程序设计语言 简体中文版 (kaisery.github.io)](https://kaisery.github.io/trpl-zh-cn/title-page.html)

### 智能指针

rust中的智能指针和C++的一样，它包了一个指针同时带了一起基本功能和属性，例如引用计数。其实`String`和`Vec<T>`也是智能指针，因为他们也拥有一块可以操作的内存。

引用Borrow它指向的数据，引用不能改变所有权。智能指针拥有它指向的数据

智能指针也使用struct来实现，只是会实现`Deref`和`Drop`两个traits。

### Box<T>

`Box<T>`指向堆上的数据的智能指针。使用`Box<T>`有三种场景

* 编译期无法获取数据大小的数据类型
* 大块的数据转移所有权，但又不想拷贝这些数据，提高性能
* 拥有一个数据时，只关心它实现的traits而不是具体的什么类型


#### Cons List

cons list是来源于Lisp语言的链表数据结构，这个链表中有两个元素，第一个元素是数据，第二个是下一个链表的元素。这个名字来源于`cons function(construct function)`在Lisp使用两个参数构造(cons)一对值(pair)，这两个参数又分别是值和另一个pair。

例如`(1, (2, (3, Nil)))`就是有三个元素的链表。linux中的` struct list`其实和这个一样，都是在list的结构中包含了下一个list的元素。

例如定义一个链表枚举

```rust
enum List {
    Cons(i32, List),
    Nil,
}

let list = Cons(1, Cons(2, Cons(3, Nil)));
```

当定义一个`let list = Cons(1, Cons(2, Cons(3, Nil)));`这样的链表时，由于链表中元素的第二个成员是另一个list，而下一个list里面又包含了一个list，编译器无法推导出这个list变量到底占用多少空间，会提示错误。此时可以将第二个成员改为Box类型，把数据放在堆上，因为Box的大小是固定的，所以编译器就可以推导出list变量占用大小。

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use crate::List::{Cons, Nil};

fn main() {
    let list = Cons(1, Box::new(Cons(2, Box::new(Nil))));
}
```

### Deref Trait

`Deref`定义了智能指针解引用的行为。一个常规的引用类型可以看作指向存储在某个地方的值的指针。我们可以使用`*`来获取引用指向的值。使用`Box<T>`可以达到和引用相同的效果

```rust
fn main() {
    let x = 5;
    let y = &x; // y的类型是&i32
    let z = Box::new(x); // z的类型是Box<i32>

    assert_eq!(5, x);
    assert_eq!(5, *y); // 使用*获取y指向的值
    assert_eq!(5, *z); // 使用*获取z指向的值
}
```

#### 自定义deref

对于自定义类型，可以通过实现`Deref`让rust使用`*`解引用一个数据。rust会把`*y`替换为`*(y.deref())`，这里的`*`替换只会工作一次，而不会把替换后的`*`再次进行替换。

```rust
use std::ops::Deref;

struct MyBox<T>(T); // 只包含了一个值的元组结构

impl<T> MyBox<T> {
   fn new(x: T) -> MyBox<T> { // new 方法创建一个对象
        MyBox(x)
   } 
}

impl<T> Deref for MyBox<T> { // 实现Deref Trait
    type Target = T; // 声明一个T的关联类型
    fn deref(&self) -> &Self::Target { 
        &self.0 // 这里返回的是引用而不是值，使用0获取元组结构的第一个值，同时不把这个值从结构中移出去move
    }
}


fn main() {
    let x = 5;
    let y = MyBox::new(x);

    assert_eq!(5, x);
    assert_eq!(5, *y); // 如果不实现Deref，会编译错误
}
```

#### 函数和方法中的隐式解引用规则

*Deref coercion*t特性可以把一个实现了`Deref` trait的引用类型转换为另一个类型的引用。例如把一个`&String`类型的参数值传递给一个需要`&str`的函数，因为`&String`的`Deref` 返回一个`&str`，所以这种调用就是可行的。这样函数和方法中的传入参数就不需要明确写`*`或`&`.当一个类型实现了`Deref` trait，rust编译器会调用调用尽可能多次的`Deref::deref`来让传入的参数引用去匹配函数需要的参数类型，这个执行过程在编译期完成，所以不会有性能影响。

```rust
fn hello(name: &str) {// 以&str为参数的函数
    println!("Hello, {name}!!!");
}

let m = MyBox::new(String::from("world"));
hello(&m); // MyBox<String>的引用会自动Deref为&String，编译器会再次调用Deref把&String转换为&str
```

#### 可变引用的解引用

使用`DerefMut` trait来实现*mutable*引用的解引用

#### 基本规则

- 当T实现了`Deref`trait返回`&U`类型，那么编译器会把 `&T` 转变为 `&U` 
- 当T实现了`DerefMut`trait返回`&mut U`类型，那么编译器会把  `&mut T`转变为 `&mut U`
- 当T只实现了`Deref`trait返回`&U`类型，那么编译器会把 `&mut T`  转变为 `&U` 

### Drop Trait

当一个变量执行出它的作用域后，会执行这个类型的`Drop` trait。例如`Box<T>`类型的变量越过它的作用域后，就会释放堆上的数据。

```rust
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        println!("Dropping CustomSmartPointer with data `{}`!", self.data);
    }
}

fn main() {
    let c = CustomSmartPointer {
        data: String::from("my stuff"),
    };
    let d = CustomSmartPointer {
        data: String::from("other stuff"),
    };
    println!("CustomSmartPointers created.");
}
```

在main函数执行结束时，会先输出变量d的Drop，再输出变量c的Drop。

#### 强制调用Drop

有时需要在出作用域之前提前释放资源，就需要提前执行drop，例如多线程使用的lock，需要在函数执行结束前就释放。但是rust不支持显式调用drop，主要为了避免多次释放资源，此时需要使用`std::mem::drop`函数。

```rust
let c = CustomSmartPointer {
        data: String::from("my stuff"),
    };
println!("CustomSmartPointer created.");
drop(c);
println!("CustomSmartPointer dropped before the end of main.");
```

### Rc<T>

### RefCell<T>

