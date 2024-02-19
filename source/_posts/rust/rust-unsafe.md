---
title: Rust Learning-Unsafe Rust
date: 2024-02-19 08:58:49
categories:
- programming
tags:
- rust
- learning
---

## Unsafe Rust

[Rust 程序设计语言 - Rust 程序设计语言 简体中文版 (kaisery.github.io)](https://kaisery.github.io/trpl-zh-cn/title-page.html)

### Unsafe Rust

unsafe rust不会强制保证内存安全，但是可以提供更强大的功能。通过使用unsafe标识，可以方便确认程序中可能有问题的代码块。

编译器有时无法判断程序正确性，所以会严格按语法规范编译失败，这时可要告诉编译器我们自己来保证程序的正确性。

使用**unsafe**关键字开始一个代码块，其中的代码可以是unsafe的，在其中可以进行以下操作：

* 解引用一个原始指针
* 调用一个unsafe的函数或方法
* 访问或修改不可变静态变量
* 实现unsafe的trait
* 访问union S的字段

#### 基本用法

##### 原始指针raw pointer

原始指针分为可变`*mut T`和不可变两种 `*const T`  ，其中的`*`号是数据类型名称的一部分，不是解引用操作。它与引用或智能指针的差异：

* 可变和不可变原始指针可以指向同一个内存地址，不需要考虑借用规则
* 不保证指向的内存地址是有效，可以访问的
* 指针的值可以是null
* 没有自动释放机制

原始指针主要用在提高程序性能，与其他语言交互或者操作硬件时。

使用**as**关键字把一个引用转换为对应的原始指针类型. rust编译器不会检查指针指向地址的有效性，两个变量同时指向同一个地址可能出现数据竞争的多线程问题。

```rust
fn main() {
    // 定义原始指针不解引用，代码都是safe的
    let address = 0x012345usize;
    let r = address as *const i32;
    
    let mut num = 5;
    // 可以同时指向相同的变量地址
    let r1 = &num as *const i32;
    let r2 = &mut num as *mut i32;

    unsafe {
        // 只能在unsafe代码块中解引用原始指针
        println!("r2 is: {}", *r2);
        *r2 = 10;
        println!("r1 is: {}", *r1);

    }
}
//r2 is: 5
//r1 is: 10
```

##### unsafe函数

使用unsafe关键字开头修饰的函数或方法只能在unsafe代码块中被调用

```rust
unsafe fn dangerous() {}

fn main() {
    //dangerous(); 
    unsafe {
        dangerous();
    }
}
```

##### 使用safe抽象来包装unsafe代码

如果一个函数中的部分代码是unsafe的，不一定要求这个函数式unsafe的。实现一个功能时需要使用unsafe的代码，例如下面的例子中从指定的索引位置分割一个数组。如果直接使用` (&mut values[..mid], &mut values[mid..]) `来返回分割的两个部分数据，编译器会认为我们同时创建了values的两个可变引用，导致出错。这时可以使用unsafe的原始指针来分割这个values的可变引用。

```rust
use std::slice;

fn split_at_mut(values: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = values.len();
    let ptr = values.as_mut_ptr(); // 获取slice的原始指针，这里的类型为*mut i32
    assert!(mid <= len); // 确保数据合法
    unsafe {
        (// 返回的元组
            // 这是个unsafe方法，需要发在unsafe代码块中，第一个参数是slice的raw point，创建一个新的slice
            slice::from_raw_parts_mut(ptr, mid), 
            slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}

fn main() {
    let mut v = vec![1, 2, 3, 4, 5, 6];
    let r = &mut v[..];
    let (a, b) = split_at_mut(r, 3);
    assert_eq!(a, &mut [1, 2, 3]);
    assert_eq!(b, &mut [4, 5, 6]);
}
```

##### 使用其他程序语言接口

rust使用**extern**关键字来创建和使用外部函数接口Foreign Function Interface(EFI).

调用外部接口时，需要在extern后面定义外部接口使用的应用二进制接口application binary interface(ABI).ABI定义了在汇编层次如何调用一个函数接口。例子中`"C"`就说明了使用C语言的ABI。

使用extern的函数都是unsafe的，因为rust无法保证外部接口的安全性。

```rust
extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    unsafe {
        println!("Absolute value of -3 according to C: {}", abs(-3));
    }
}
```

##### 提供外部语言使用的rust接口

同理可以让外部语言使用rust实现的接口。`#[no_mangle]`用来让编译器不要对函数名进行混淆，避免外部调用时在库中找不到函数，同样也需要用extern关键字后加ABI类型指明调用的接口类型。

```rust
#[no_mangle]
pub extern "C" fn call_from_c() {
    println!("Just called a Rust function from C!");
}
```

##### 访问或修改可变静态变量

rust中的全局变量称为静态变量。静态变量和常量类似，也使用 `SCREAMING_SNAKE_CASE`  命名习惯。使用**static**关键字修饰，rust编译器可以明确静态变量的声明周期。

静态变量和常量的差异：

1. 静态变量在内存中有固定的地址，静态变量也可以定义为可变的
2. 常量在使用的地方都有一份复制

访问不可变静态变量是safe的，但是读或写可变静态变量都是不安全的，都需要在unsafe代码块中，因为可能存在多线程的数据竞争问题。

```rust
static mut COUNTER: u32 = 0;

fn add_to_count(inc: u32) {
    unsafe {
        COUNTER += inc;
    }
}

fn main() {
    add_to_count(3);
    unsafe {
        println!("COUNTER: {}", COUNTER);
    }
}
```

##### Unsafe Trait

一个Trait中的方法如果有编译器无法验证的不变体，这个Trait就是不安全的，实现这个trait时也需要声明unsafe。

```rust
unsafe trait Foo {
    // methods go here
}

unsafe impl Foo for i32 {
    // method implementations go here
}

fn main() {}
```

##### 联合体中的字段

rust的中联合体和C中的类似，一个时刻只能使用union中定义的一个字段，主要也是为了和C语言交互使用的，但是rust编译器无法确定当前union中的成员具体的数据是什么样的，所以访问union中的字段也是不安全的。