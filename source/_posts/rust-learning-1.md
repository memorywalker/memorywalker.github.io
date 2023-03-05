---
title: Rust Learning 1
date: 2023-03-05 09:25:49
categories:
- programming
tags:
- rust
- learning
---

## RUST Learning 1

[Rust 程序设计语言 - Rust 程序设计语言 简体中文版 (kaisery.github.io)](https://kaisery.github.io/trpl-zh-cn/title-page.html)


### 所有权(Ownership)


#### 规则

1. 每一个值都有一个所有者(owner)
2. 值在任何时刻只能有一个所有者
3. 当所有者(变量)离开作用域，这个值就被释放

rust中的作用域和C的一样。

#### 资源释放

以String类型为例，一个String类型变量值存储在栈上，但是它实际指向的字符串数据内存在堆上。

![string_pointer](../uploads/rust/string_pointer.png)

```rust
    {
        let s = String::from("Flower");
    } // s drop
```

当变量s离开作用域，rust会调用drop函数来释放内存。这个机制类似C++中的Resource Acquisition Is Initialization(RAII)，一个对象在生命周期结束时，自己释放拥有的资源。

##### 移动

变量的所有权规则：将值赋给另一个变量时移动它，当持有堆中的数据的变量离开作用域时，其值通过drop被清理掉，除非数据被移动为另一个变量所有。

```rust
    {
        let x = 5;
        let y = x;
        println!("x is {x} y is {y}");
        let s1 = String::from("Flower");
        let s2 = s1;
        println!("s2 is {s2} s1 is {s1}"); // error: borrow of moved value: `s1`
    } 
```

对于复杂的数据类型，变量之间在赋值时，相当于把前一个变量s1**移动**到了s2，这样避免了s1和s2都还指向子串的实际内容，退出作用域时，s1和s2都会对内存资源进行释放导致double free。对于普通的数据类型，rust给x和y在栈上各提供了一个5作为值。



##### 克隆

rust永远不会自动创建数据的深拷贝。

如果需要深度复制String在堆上的数据，可以使用clone函数。clone出现的地方说明有额外的代码执行可能会很耗资源。

```rust
let s1 = String::from("Flower");
let s2 = s1.clone();
println!("s2 is {s2} s1 is {s1}");
```

Rust有个Copy trait的特殊注解，如果一个类型实现了Copy trait，那么一个旧的变量将其赋给其他变量后仍然可用。基本的整数类型，bool类型，浮点类型，字符类型，以及只包含实现了Copy元素的元组类型都是Copy类型。

Rust禁止自身或其任何部分实现了Drop trait的类型使用Copy trait。

##### 函数参数

对于不支持Copy的类型作为参数，会把传入参数的变量移动到函数内，除非把这个变量通过函数返回出来，否则之前的变量由于被移动走，无法使用。

```rust
fn take_owner(str: String) {
    println!("func string: {}", str);
} // str 退出作用域调用drop，把字串占用的内存资源释放

fn make_copy(value: i32) {
    println!("func integer: {}", value);
} 

	let s1 = String::from("Flower");
    take_owner(s1); // s1 moved into function
    // s1 is not valid here
    let x = 5; 
    make_copy(x);  // copy for i32 type
    println!("integer: {}", x);  // x is still valid   
```

##### 函数返回值

函数的返回值可以把函数内的变量的所有权移动给函数外的变量。

```rust
fn give_owner() -> String {
    let game = String::from("call of duty");
    game  // 注意这里没有语句结束；所以作为一个表达式返回变量game
}
let fps = give_owner(); // 变量的所有权现在归fps
```

##### 引用

如果一个变量作为参数把值的所有权移动到了函数体内，函数执行后还需要使用这个变量的地方就不能使用这个变量了，如果每次把参数再作为返回值把所有权移动出来也会很麻烦。此时可以使用引用作为函数的参数。

引用像一个指针，它是一个地址，我们可以由此访问存储于该地址属于其他变量的数据。引用需要确保它指向了某个特定类型的有效值。

创建一个引用的行为称为借用(borrowing)

```rust
fn cal_str_len(s: &String) -> usize {
    s.len(); // 引用使用值，但不获取所有全，但是默认不能修改值
}
let s1 = String::from("Flower");
let len = cal_str_len(&s1); //使用引用作为参数
println!("string {} len is {}", s1, len);  // s1还有所有权
```

###### 可变引用

通过使用mut关键字可以声明一个引用是可修改的。

```rust
fn change_ref(str: &mut String) {
    str.push_str(" is beautiful"); // 修改一个引用
}
let mut s1 = String::from("Flower"); // 定一个可变字符串
change_ref(&mut s1);  // 可变引用参数
println!("string {}", s1);
```

一个引用的生命周期从这个引用定义开始，到这个引用的最后一次使用终止。

如果已经有一个对变量的可变引用，在这个引用的生命周期内，不能对被引用的变量再次引用，这样会导致多个引用修改或访问同一个变量，引发多线程的数据竞争问题。同样，不可变引用和可变引用也不能同时存在。

```rust
let mut s1 = String::from("Flower");
let r1 = &mut s1;
let r2 = &mut s1; // 编译器会提示 ^^^^^^^ second mutable borrow occurs here
println!("{} {} ", r1, r2); // -- first borrow later used here
```

如果对一个变量的引用都是不可变的，那么不存在数据竞争访问问题，是可以使用的。

Rust的编译器会保证一个引用不会变成**悬垂引用(Dangling Reference)**.

```rust
fn dangle_ref() -> &String { // 返回一个字符串引用
    let s = String::from("Flower");
    &s // 返回引用
} // s 退出作用域，内存资源被释放
编译器提示：
this function's return type contains a borrowed value, but there is no value for it to be borrowed from
```

总结：

* 要么只能有一个可变引用，要么只有多个不可变引用
* 引用必须总是有效的

##### Slice类型

slice是一种引用，所以它没有所有权。可以引用集合中一段连续的元素序列，是一个部分不可变引用。

```rust
let poem = String::from("best way to find a secret");
let key = &poem[0..4];
```

`[start..end]`表示从start开始，end-start长度的子集。当start为0时，可以不写，end为最后一个字符时也可以省略。

字符串slice的类型声明为`&str`

```rust
fn fisrt_word(s: &String) -> &str { // 返回一个String的slice
    let bytes = s.as_bytes(); // 转换为字符数组
    for (i, &item) in bytes.iter().enumerate() { // 数组迭代器
        if item == b' ' { // 找到第一个空格的位置
            return &s[0..i]; // 截取第一个空格之前的字符为第一个字
        }
    }
    &s[..]  // 没有空格
}

```

`let s = "book a ticket";`中s的类型是`&str`，他是指向一个二进制程序特定位置的slice，由于他是一个不可变引用，所以值不可改变。

对于一个整型数的数组他的slice数据类型为`&[i32]`