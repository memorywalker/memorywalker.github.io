---
title: Rust Learning-Object Oriented Programming
date: 2024-02-17 09:42:49
categories:
- programming
tags:
- rust
- learning
---

## RUST Object-Oriented Programming

[Rust 程序设计语言 - Rust 程序设计语言 简体中文版 (kaisery.github.io)](https://kaisery.github.io/trpl-zh-cn/title-page.html)

### 面向对象编程

 Object-oriented programs are made up of objects. An *object* packages both data and the procedures that operate on that data. The procedures are typically called *methods* or *operations*. 

### Rust中的面向对象

rust中的struct和enum可以定义不同的数据结构，并可以给结构定义的方法

#### 封装隐藏实现

rust中使用`pub`关键字来控制数据结构访问，例如定义一个计算平均值的结构体，数据成员为私有，添加和删除方法为公开的，每次添加新的数据时自动调用计算平均值私有方法计算出平均值。

```rust
pub struct AveragedCollection {
    list: Vec<i32>,  // 外部不能直接访问
    average: f64,
}

impl AveragedCollection {
    pub fn add(&mut self, value: i32) {
        self.list.push(value);
        self.update_average();
    }

    pub fn remove(&mut self) -> Option<i32> {
        let result = self.list.pop();
        match result {
            Some(value) => {
                self.update_average();
                Some(value)
            }
            None => None,
        }
    }

    pub fn average(&self) -> f64 {
        self.average
    }

    fn update_average(&mut self) {
        let total: i32 = self.list.iter().sum();
        self.average = total as f64 / self.list.len() as f64;
    }
}
```

当外部程序使用这个结构体时，不需要知道其中数据是怎么组织的，只需要调用添加、删除和平均值三个公开接口。如果这个结构内部数据结构调整或更新计算平均值的规则，外部使用者不会被影响。

#### 类型继承实现代码复用

rust中的struct不支持父子继承关系，如果一定要复用接口，可以通过trait的方法默认实现，让struct声明支持一个trait的方法，这个方法在trait中已经提供了默认实现。

继承在现在很多编程语言中已经不是主流的编程范式，因为继承共享了太多不需要的实现，有的语言只支持单继承。但是我现在主要开发工作中面相对象还是最主要的编程方法，抽象，多态使用的还是很多的。

### Trait Object

一个trait object同时指向一个实现了某个具体trait的实例和一个在运行时用来查找类型中trait方法的表格。trait object的声明需要一个指针如`&引用`或`Box<T>`并在trait类型前加上`dyn`关键字。Trait object作为泛型或具体类型使用。rust编译器会保证对应的实例实现了trait的方法。

例如 `Box<dyn Draw> `就是一个trait object，它表示在一个Box中的实现了Draw这个trait的任意类型。

下面的例子中假设gui库中有个Draw Trait，gui库中有个screen结构体，它的run方法调用每一个控件的draw方法。库默认提供了button控件。使用gui库的应用程序中可以自己定义一个SelectBox控件，它实现了Draw Trait，所以即使它并没有在库中定义，也可以加在screen的控件列表中被执行。

```rust
pub trait Draw {// 定义一个有draw方法的trait
    fn draw(&self);
}

pub struct Screen {
    // screen结构中有多个可以绘制的控件列表，列表中的都是trait object
    pub components: Vec<Box<dyn Draw>>,
}

impl Screen {
    // run方法依次调用每一个控件对象执行它的draw方法
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
// lib库中定义了一个button控件，实现了Draw Trait
pub struct Button {
    pub width: u32,
    pub height: u32,
    pub label: String,
}

impl Draw for Button {
    fn draw(&self) {
        println!("draw a button!");
    }
}
// 用户应用程序自定义控件，同样实现了Draw Trait
struct SelectBox {
    width: u32,
    height: u32,
    options: Vec<String>,
}

impl Draw for SelectBox {
    fn draw(&self) {
        println!("draw a SelectBox!");
    }
}

fn main() {
        let screen = Screen {
        components: vec![
            Box::new(SelectBox {
                width: 75,
                height: 10,
                options: vec![
                    String::from("Yes"),
                    String::from("Maybe"),
                    String::from("No"),
                ],
            }),
            Box::new(Button {
                width: 50,
                height: 10,
                label: String::from("OK"),
            }),
        ],
    };

    screen.run();
}
```

#### 与模版差异

对于上面的screen的例子如果使用模版来实现

```rust
pub struct Screen<T: Draw> {
    pub components: Vec<T>,
}

impl<T> Screen<T>
where
    T: Draw,
{
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```

1. 模板每次只能具体化一个类型，例如`Screen<Button>`那么其中的控件就只能全部都是button，对于trait object就可以支持不同的类型。
2. 对于模板，编译器在编译期就可以为每个类型生成对应的静态代码，而trait object是动态派发，rust在运行时通过trait object的方法指针来决定调用的方法，就存在方法查找的损耗，同时静态编译的方法中内联优化也无法在动态派发中支持，所以使用trait object的性能会差异性，但是更灵活。

### rust实现面向对象设计模式

