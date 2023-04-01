---
title: Rust Learning-Generic, Trait and Lifetimes
date: 2023-04-01 22:42:49
categories:
- programming
tags:
- rust
- learning
---

## RUST 

[Rust 程序设计语言 - Rust 程序设计语言 简体中文版 (kaisery.github.io)](https://kaisery.github.io/trpl-zh-cn/title-page.html)

### 泛型

把函数，结构体中变量的类型参数化，所以T类似于表示数据类型的形参。T是type的缩写，和C++一样大家习惯用T来代表一种类型。

#### 函数中泛型

如果要使用一个表示类型的参数，需要在使用前声明，所以在函数的名称和参数列表中间使用<>进行类型参数的声明。

```rust
fn largest<T: std::cmp::PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
let number_list = vec![1,5,67,82,34,22];
let result = largest(&number_list);
```

由于在函数中对T类型进行了比较操作，所以T类型必须是支持比较`std::cmp::PartialOrd`的。

#### 结构体中泛型

可以定义多个泛型类型，例如我们可以给结构体中不同成员使用不同的类型。

```rust
struct Point<T, U> {
    x: T,
    y: U,
}
// x 和 y是不同的数据类型
let int_float_value = Point {x:5, y:5.0};
```

#### 枚举中泛型

枚举中的每一个值可以是不同的泛型类型。

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

#### 方法中泛型

impl后使用<>声明结构体的泛型参数，例如下例中`impl<T, U>`说明了`Point`后面的`<T, U>`是泛型参数，而不是具体的类型。这里`impl<T, U> Point<T, U>`中使用的泛型参数必须一致。但是可以与结构体声明时使用的泛型参数不同。

`fn mixup<X, Y>`中方法名后的泛型参数说明这个方法中要使用的泛型参数，它的使用范围在这个方法内部。

`impl Point<f32, f32>`表示给具体的f32类型的Point定义的方法，其他类型的Point则没有这个方法。

```rust
impl<T, U> Point<T, U> {
    fn mixup<X, Y>(self, other: Point<X, Y>) -> Point<T, Y> {
        Point {
            x: self.x,
            y: other.y,
        }
    }
}
impl Point<f32, f32> {
    fn distance_from_origin(&self) ->f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

#### 泛型性能

编译器会查看所有泛型代码被使用的地方，根据使用的上下文推导出泛型代表的实际类型，生成对应具体类型的代码，在调用的地方实际调用的是编译器生成的具体类型的函数，结构体或枚举。和C++的原理一样，因为不是运行时的行为，所以不存在性能损耗。

### Trait

Trait定义了一组不同类型拥有共同的方法。类似于Java中的接口，定义的trait就像定一个接口，但又略有不同。

例如书和游戏都有获取总结信息的方法，时间类型和日期类型都有输出格式化字符串的方法。这些方法就像是接口中声明的方法，哪个类型支持这个功能，只需要**实现**这个方法，外部就可以使用这个类型的这个功能。

如下定义了一个名称为Summary的Trait，它声明了一个summarise的方法，如果一个类型支持这个Trait功能，它需要实现这个方法。类似具体类型要实现接口的的方法，来支持接口。

```rust
pub trait Summary {
    fn summarise(&self) -> String; // 这里没有具体的实现，类似纯虚接口
}
```

**一个类型实现一个Trait**

```rust
#[derive(Debug)]
struct Game {
    game_name: String,
    game_type: GameType,
    rate: f32,
}

#[derive(Debug)]
enum GameType {
    FPS,
    RPG,
    Sport,
}

impl Summary for Game { // 为Game类型实现Summary这个Trait
    fn summarise(&self) -> String {
        format!("{} is a {:?} game.", self.game_name, self.game_type)
    }
}

let cod = Game {
    game_name:String::from("Call of Duty"),
    game_type:GameType::FPS,
    rate:6.0,
};
println!("Game info: {}", cod.summarise()); // Game info: Call of Duty is a FPS game
```

* 当要实现Trait的类型位于他自己的Crate本地作用域时，可以为它实现Trait，例如自定义的Game结构所在的Crate中可以为Game实现标准库中的Display trait。
* 在一个Trait声明的Crate作用域中，可以给其他Crate中的类型实现这个Trait，例如可以在自己定义的Summary trait的Crate中为标准库的`vec<T>`实现Summary trait。

但是不能为外部类型实现trait，那样外部使用库的人就可以修改库的行为，相当于破坏库的代码了，rust也无法判断要执行谁的实现。

#### Trait默认实现

可以像抽象方法实现接口那样给Trait的方法提供默认实现，这样其他类型只需要声明他实现了这个trait，而不需具体方法体实现。

```rust
pub trait Summary {
    fn summarise(&self) -> String { // 默认实现一个方法
        format!("This is {}.", self.my_type()) // 可以调用这个Trait中的其他方法
    }

    fn my_type(&self) -> String;
}

impl Summary for Game { // 具体类型中不需要实现有默认实现的summarise方法了
    fn my_type(&self) -> String { // 没有默认实现的方法还必须实现
        String::from("Game")
    }
}
```

#### Trait作为参数

有点像把接口类型作为函数形参，实参使用实现了这个接口的具体对象。参数的类型需要关键字**impl**

```rust
fn notify(item: &impl Summary) {// item的类型为实现了Summary这个trait的所有类型
    println!("Notify {}", item.summarise());
}
notify(&cod); // Notify This is Game.
```

##### Trait Bound

上面Summary作为参数的完整写法为

```rust
fn notify<T: Summary>(item: &T) {
    println!("Notify {}", item.summarise());
}
// fn notify(item: &impl Summary, item2: &impl Summary) {
fn notify2<T: Summary>(item: &T, item2: &T) { // 每个参数的类型写法简单了一点
    println!("Notify {} and {}", item.summarise(), item2.summarise());
}
```

这种使用泛型的表示方法称为trait bound。当如果有多个参数，且参数类型相同时，就可以简化函数的声明。

##### 同时使用多个Trait

使用`+`把多个trait连起来

```rust
fn notify(item: &(impl Summary + std::fmt::Display)) {
    println!("Notify {}", item.summarise());
}
fn notify<T: Summary + std::fmt::Display>(item: &T) { // trait bound写法
    println!("Notify {}", item.summarise());
}
```

##### 使用where优化写法

在where中统一描述泛型类型的Trait

```rust
fn notify2<T, U>(item: &T, item2: &U) 
where 
    T: Summary + fmt::Display,
    U: Summary + fmt::Debug,
{
    println!("Notify {} and {}", item.summarise(), item2.summarise());
}
```

#### Trait作为返回值

返回值类型为`impl trait_name`.使用Trait作为返回值类型时，只能返回一种具体类型，不能返回实现了Trait的多种不同具体类型。

```rust
fn new_summarizable(name: String) -> impl Summary {
    Game {
        game_name:name,
        game_type:GameType::FPS,
        rate:6.0,
    }
}
```

#### 使用Trait Bound有条件的实现方法

这个语法主要用在编写库程序，对于使用了泛型定义的类型，可以限制实现了指定Trait的类型才提供方法。

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T: Display + std::cmp::PartialOrd> Point<T>  { // 实现了Display和PartialOrd的类型才能调用这个方法
   fn cmp_display(&self) {
       if self.x >= self.y {
            println!("Left");
       }
       else {
            println!("Top");
       }
   } 
}
let int_point = Point {x:5, y:10};// i32实现了Display和PartialOrd，所以可以调用
int_point.cmp_display();
```

对任何满足特定Trait Bound的类型实现的trait称为blanket implementations. 标准库中给所有实现了Display和Size的类型实现了ToString这个Trait。这个Trait里面只有一个`to_string()`的方法。

```rust
impl<T: fmt::Display + ?Sized> ToString for T {
```

```rust
// 让Game实现Display
impl std::fmt::Display for Game {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "({}, {:?})", self.game_name, self.game_type)
    }
}

// 给所有实现了Display的类型实现Summary
impl<T: Display> Summary for T {
    fn my_type(&self) -> String {
        self.to_string()
    }
}
```



### 生命周期

