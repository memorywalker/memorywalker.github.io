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

#### 状态模式

状态模式在状态内部封装数据，数据或行为会根据状态而不同。每一个状态只处理自己支持的行为和如何切换到其他状态。状态对象的拥有者不需要知道状态如何切换。当业务发生变化时，只需要更新状态内部的代码或增加新的状态，而不用更改拥有状态的业务代码。

一个博客文章分为草稿、审阅、发布几个阶段，每个阶段有自己可以支持的操作，不同的阶段之间可以转换。

1.  一个博客文章Post有内容和当前的状态
2. Post默认为空的草稿状态
3. Post添加内容后，直到发布前外部看到都是空内容，所以通过状态来确定Post的Content是什么

使用到的技术要点：

1. 使用new方法来创建对象，并进行基本的初始化
2. state trait的方法使用`Box<Self>`作为参数，并返回一个trait object`Box<dyn State>`
3. post使用state来处理返回的content时，把post作为引用传入方法，但是返回值又是post的成员，需要使用生命周期注解说明返回值的生命周期和入参post的生命周期相关

```rust
pub struct Post {
    state: Option<Box<dyn State>>,
    content: String,
}

impl Post {
    pub fn new() -> Post {
        Post {
            state: Some(Box::new(Draft {})),// 默认创建一个空的草稿状态
            content: String::new(),
        }
    }
    // 添加内容
    pub fn add_text(&mut self, text: &str) {
        self.content.push_str(text);
    }
    
    pub fn content(&self) -> &str {
        // as_ref()返回Option<&Box<dyn State>>使用引用，因为不能把state的所有权从post结构 move走
        self.state.as_ref().unwrap().content(self)
    }
    
    pub fn request_review(&mut self) {
        if let Some(s) = self.state.take() {
            // 调用当前状态的request_review，request_review方法会获取s的所有权
            // rust要求结构体的成员必须有值，所以使用request_review返回的状态
            // 重新赋值给Post的state，达到状态的切换
            self.state = Some(s.request_review())
        }
    }
    
    pub fn approve(&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.approve())
        }
    }
}

// 所有状态支持的行为
trait State {
    // self的类型为Box<Self>，只有在一个Box<T>类型对象上调用这个方法才有效，
    // 这个参数的所有权传入方法，并返回一个新的相同类型的状态对象 
    fn request_review(self: Box<Self>) -> Box<dyn State>;
    
    fn approve(self: Box<Self>) -> Box<dyn State>;
    // 默认实现返回空
    // 这里使用了声明周期注解，因为post作为引用传入方法，但是方法的返回值
    // 又是post这个引用的成员，所以需要告诉编译器返回值的生命周期和入参
    // post的一致
    fn content<'a>(&self, post: &'a Post) -> &'a str {
        ""
    }
}

struct Draft {}

impl State for Draft {
    fn request_review(self: Box<Self>) -> Box<dyn State> {
        Box::new(PendingReview {})
    }
    
    fn approve(self: Box<Self>) -> Box<dyn State> {
        self
    }
}

struct PendingReview {}

impl State for PendingReview {
    fn request_review(self: Box<Self>) -> Box<dyn State> {
        self
    }
    // 审阅后的文章转换为发布状态
    fn approve(self: Box<Self>) -> Box<dyn State> {
        Box::new(Published {})
    }
}

struct Published {}

impl State for Published {
    fn request_review(self: Box<Self>) -> Box<dyn State> {
        self
    }

    fn approve(self: Box<Self>) -> Box<dyn State> {
        self
    }
    
    fn content<'a>(&self, post: &'a Post) -> &'a str {
        &post.content
    }
}

fn main() {
    let mut post = Post::new();

    post.add_text("I ate a salad for lunch today");
    assert_eq!("", post.content());

    post.request_review();
    assert_eq!("", post.content());

    post.approve();
    assert_eq!("I ate a salad for lunch today", post.content());
    
    println!("All work done!!!");
}
```

#### 利弊

优点：

1. 方便扩展新的状态，例如增加一个驳回操作，或者需要两次审阅才能发布
2. 不需要很多的match分支判断

缺点：

1. 状态之间存在依赖，一个状态切换下一个状态的规则
2. 状态实现了公共接口Trait重复的代码
3. Post需要委派相同的方法给state，例如 approve 方法

#### 状态和行为定义为类型

除了使用面相对象的方式实现一个功能，还可以利用rust语言的特有机制实现相同的功能，面相对象不是唯一的方案。

rust编译器的类型检查可以帮助我们检查一个对象支持哪些操作，例如草稿状态下不能返回内容，只能进行审阅。

rust的所有权转移可以通过方法调用让一个类型转换为另一个类型的对象，例如：

1. Post默认new出来的是DraftPost对象
2. DraftPost对象有添加内容方法和请求审阅方法，请求审阅方法会返回一个PendingReviewPost对象
3. PendingReviewPost对象执行它特有的approve方法，返回一个Post对象

```rust
pub struct Post {
    content: String,
}

pub struct DraftPost {
    content: String,
}

impl Post {
    pub fn new() -> DraftPost {
        DraftPost {
            content: String::new(),
        }
    }

    pub fn content(&self) -> &str {
        &self.content
    }
}

impl DraftPost {
    pub fn add_text(&mut self, text: &str) {
        self.content.push_str(text);
    }
    
    pub fn request_review(self) -> PendingReviewPost {
        PendingReviewPost {
            content: self.content,
        }
    }
}

pub struct PendingReviewPost {
    content: String,
}

impl PendingReviewPost {
    pub fn approve(self) -> Post {
        Post {
            content: self.content,
        }
    }
}

fn main() {
    let mut post = Post::new();

    post.add_text("I ate a salad for lunch today");
	// 所有权转移了，所以需要新的变量
    let post = post.request_review();

    let post = post.approve();

    assert_eq!("I ate a salad for lunch today", post.content());
    
    println!("All works done!!!");
}
```

