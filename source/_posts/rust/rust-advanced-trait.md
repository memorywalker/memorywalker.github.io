---
title: Rust Learning-Advanced Traits and Types
date: 2024-02-19 11:36:49
categories:
- programming
tags:
- rust
- learning
---

## Advance Traits

[Rust 程序设计语言 - Rust 程序设计语言 简体中文版 (kaisery.github.io)](https://kaisery.github.io/trpl-zh-cn/title-page.html)

### 关联类型

关联类型(associated types)是用一个类型占位符和trait关联的实现方法，在trait的方法声明中可以使用这些占位符类型，trait的实现者需要在实现时指定这个占位符类型的实际具体类型。

例如标准库的 `Iterator`  trait有个Item的关联类型来替代遍历的值类型，它的next方法中也能使用这个类型。

```rust
pub trait Iterator {
    type Item;// 关联类型

    fn next(&mut self) -> Option<Self::Item>;//使用关联类型
}
```

实现trait时需要说明关联类型具体是什么

```rust
impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
    }
}
```

如果一个trait有泛型参数，那么这个trait就可以有很多个不同的类型实现，在给一个结构体实现一个trait时，就需要指明实现的是哪个类型的trait，例如 `Iterator<String> for Counter `，而使用关联类型就不需要指明具体哪个类型的trait，因为这个trait只有一种实现。

### 默认泛型类型参数

当使用泛型类型参数时，可以为泛型指定一个默认的具体类型。 <PlaceholderType=ConcreteType> 。

使用默认参数类型主要解决两类问题（和实际工作中C++的类似）：

* 需要调整接口的参数类型，而不想影响现有代码，所以给接口声明一个默认的参数类型
* 大部分情况下使用一种默认的类型就足够了，偶尔使用特殊的某个类型的参数

#### 运算符重载

rust不允许直接重载运算符，但是可以通过`std::ops`中支持的运算符和对应的trait实现运算符重载。例如可以为`Point`类型实现`Add` Trait来重载`+`运算符。

`Add` trait的声明如下，它有一个泛型类型参数Rhs，默认这个类型就是类型自己Self。

```rust
trait Add<Rhs=Self> {
    type Output;

    fn add(self, rhs: Rhs) -> Self::Output;
}
```

给Point实现这个trait，从而可以实现两个Point的直接`+`运算，默认情况下add方法的第二个参数就是类型自身，这里就是Point。

```rust
use std::ops::Add;

#[derive(Debug, Copy, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Output = Point; // 明确关联类型的具体类型为Point

    fn add(self, other: Point) -> Point {
        Point {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

fn main() {
    assert_eq!(
        Point { x: 1, y: 0 } + Point { x: 2, y: 3 },
        Point { x: 3, y: 3 }
    );
}
```

当然也有两个不同类型的对象相加的情况，例如把毫米和米进行相加。在实现trait时，指定了泛型参数类型为`Meters`，所以在相加时，第二个参数*1000后

```rust
use std::ops::Add;
#[derive(Debug, Copy, Clone, PartialEq)]
struct Millimeters(u32);
struct Meters(u32);

impl Add<Meters> for Millimeters {
    type Output = Millimeters;

    fn add(self, other: Meters) -> Millimeters {
        Millimeters(self.0 + (other.0 * 1000))
    }
}

fn main() {
    assert_eq!(
        Millimeters (200)  + Meters (1),
        Millimeters (1200)
    );
}
```

### 完全限定语法消除歧义

两个不同的trait可以有相同的方法名称，而同一个结构又可以实现多个trait，结构自身可能也存在和trait有相同名称的方法。

为了让编译器区分当前实际调用的是哪个方法实现，需要使用完全限定语法。

```rust
trait Pilot {
    fn fly(&self);
}

trait Wizard {
    fn fly(&self);
}

struct Human;

impl Pilot for Human {
    fn fly(&self) {
        println!("This is your captain speaking.");
    }
}

impl Wizard for Human {
    fn fly(&self) {
        println!("Up!");
    }
}

impl Human {
    fn fly(&self) {
        println!("*waving arms furiously*");
    }
}
fn main() {
    let person = Human;
    Pilot::fly(&person); // This is your captain speaking.
    Wizard::fly(&person); // Up!
    person.fly(); // *waving arms furiously*
}
```

由于fly是第一个参数为self的关联方法，所以可以使用trait名称前缀，并把对象传入调用的方法的调用方法，这样编译器知道是要调用哪个trait的方法，同时由于传入了具体的对象，编译器也知道要调用哪个对象的实现。

对于一些不是关联方法的函数，由于他们没有self参数，无法获取对象的类型，就只能使用完全限定语法。

```rust
<Type as Trait>::function(receiver_if_method, next_arg, ...);
```

使用`<Dog as Animal>`明确指定使用Animal的方法实现

```rust
trait Animal {
    fn baby_name() -> String;
}

struct Dog;

impl Dog {
    fn baby_name() -> String {
        String::from("Spot")
    }
}

impl Animal for Dog {
    fn baby_name() -> String {
        String::from("puppy")
    }
}

fn main() {
    println!("A baby dog is called a {}", Dog::baby_name()); // 调用Dog的方法
    println!("A baby dog is called a {}", <Dog as Animal>::baby_name()); // 调用Dog的Animal实现
}
```

### Trait之间的复用和依赖

一个trait A实现时可以使用结构体已经实现的另一个trait B的方法。这个B就是A的父trait(super trait)。

例如要实现一个格式化打印内容的trait `OutlinePrint `，在实现它的打印方法时，会用到标准库的 `Display `trait的功能，所以在实现`OutlinePrint `的时候要求这个结构也实现了`Display `trait，通过声明这两个trait的父子关系`trait OutlinePrint: fmt::Display`，就可以让编译器强制检查是否满足已经实现了被依赖的`Display`trait。

```rust
use std::fmt;

trait OutlinePrint: fmt::Display {// 指明trait的依赖关系，Display为父
    fn outline_print(&self) {
        let output = self.to_string(); // to_string是Display的方法，可以放心直接调用了
        let len = output.len();
        // 根据内容的宽度格式化整体的框的宽度
        println!("{}", "*".repeat(len + 4));
        println!("*{}*", " ".repeat(len + 2));
        println!("* {} *", output);
        println!("*{}*", " ".repeat(len + 2));
        println!("{}", "*".repeat(len + 4));
    }
}

struct Point {
    x: i32,
    y: i32,
}
// 如果Point没有实现Display，就会编译错误
impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
// 直接使用trait的默认实现就行了
impl OutlinePrint for Point {}

fn main() {
    let point = Point { x: 1000, y: 1};
    point.outline_print();
}

*************
*           *
* (1000, 1) *
*           *
*************
```

## Advanced Types

###  **newtype 模式** 

#### 在外部类型上实现外部Trait

 孤儿规则（orphan rule）：只要 trait 或类型对于当前 crate 是本地的话就可以在此类型上实现该 trait。但是如果我们要为`Vec<T>`实现`Display `Trait，由于这两个类型都在我们自己crate的外部，所以按规则是无法实现的。

 **newtype 模式**（*newtype pattern*），它使用一个元组结构体中创建一个新类型。这个元组结构体封装一个希望实现 trait 的类型的字段。这个封装的新类型对于 crate 是本地的，所以可以对它实现 trait。*Newtype* 是源自 Haskell 编程语言的概念。使用这个模式没有运行时性能损失，这个封装的新类型在编译时会被省略掉。 

```rust
use std::fmt;

struct Wrapper(Vec<String>);

impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}

fn main() {
    let w = Wrapper(vec![String::from("hello"), String::from("world")]);
    println!("w = {}", w);
}
```

在实现Display时，使用了`self.0`来访问元组结构体的唯一一个成员。这种方法的缺点是由于封装了一层新类型，我们无法直接访问原来vec的所有方法，只能通过重新对封装来实现相同的方法，来委派给内部的类型。如果封装类需要所有内部类型的方法，可以通过实现 `Deref` trait  来获取内部的类型，直接调用内部类型的方法。

#### newtype其他用途

* 静态的标识一个值不会被混淆或标识值的单位，例如下面的类型作为函数参数就可以保证有类型检查

```rust
struct Millimeters(u32);
struct Meters(u32);
```

* 通过newtype包装内部的数据类型，可以只暴露一些公共的方法给外部使用

### 类型别名

类型别名的作用和C++的typedef类似，它不会定一个一个新类型，只是给同一个类型多了一个名字。当类型的名字比较长时，可以使用这个比较短的名字作为类型名。别名的声明使用**type**关键字

例如有个很长的类型`Box<dyn Fn() + Send + 'static>` 可以给他起个别名为Trunk

```rust
type Thunk = Box<dyn Fn() + Send + 'static>;
let f: Thunk = Box::new(|| println!("hi"));
fn takes_long_type(f: Thunk) {
        // --snip--
}
```

 别名通常 和`Result<T, E>`  配合使用，减少重复的代码。在标准库的std::io中也使用了别名

```rust
type Result<T> = std::result::Result<T, std::io::Error>;
pub trait Write {
    // fn write(&mut self, buf: &[u8]) -> Result<usize, Error>;
    fn write(&mut self, buf: &[u8]) -> Result<usize>; 
    // fn flush(&mut self) -> Result<(), Error>;
    fn flush(&mut self) -> Result<()>;
}
```

### Never Type

rust中`!`被称为never type，因为它可以用来标识一个函数永远不会执行完。目前`!`只能用在函数返回值，标识这个函数是一个发散函数永远不会返回。

`!`和 `panic!` 配合使用，由于后者不会返回一个值，它会直接结束程序，所以也是一种不会返回状态。

```rust
fn foo() -> ! {
    panic!("This call never returns.");
}

extern "C" {
    pub fn no_return_extern_func() -> !;
}
```

`!`作为一个没有值类型还可以作为match的一个分支的表达式。match语句要求所有分支的返回类型都必须相同，在下面的例子中，第一个分支返回一个u32的数字，如果第二个分支返回字串，会直接报错。但是如果使用`continue`，由于它有一个`!`值，所以编译器会认为第二个分支没有值，就用第一个分支的返回值类型u32作为match的返回类型。


```rust
let guess: u32 = match guess.trim().parse() {
    Ok(num) => num,
    Err(_) => continue,
};
```

无限loop循环不会结束，所以这个表达式的值为`!`

```rust
fn foo() -> ! {
    loop {
        print!("and ever ");
    }
}
```

### 动态大小类型和Sized Trait

 *dynamically sized types*(*DSTs*) 或*unsized types* 是只有在运行时才能获取值实际占用空间的类型。

例如`str`类型就是动态类型大小的，因为只有运行时才知道这个字符串的大小。因此我们不能创建一个str类型的变量，因为编译器不知道给这个变量在内存分配多大的内存空间。rust提供了字串切片类型`&str`，它存储了这个字串的地址和字串的长度，所以`&str`类型的大小是固定已知的，可以定义`&str`类型的变量。

动态大小类型需要和一个指针配合使用，让指针类型指向动态类型数据的地址，例如使用智能指针或`&`引用。

trait也是一个动态大小类型，所以trait object需要放在一个指针中，例如`&dyn Trait`或者`Box<dyn Trait>`

rust提供了`Sized` trait来判断一个类型的大小在编译期是否是已知的。它会被每一个可以获取到大小的类型自动实现。

rust隐含的给每一个泛型函数的都使用`Sized` trait类型，泛型类型T的类型必须是已知大小的

```rust
fn generic<T: Sized>(t: T) {
    // --snip--
}
```

我们可以修改这种默认的声明，让T可以是一个不定大小的，但是参数的类型需要调整为&T，因为T的类型大小未知。

```rust
fn generic<T: ?Sized>(t: &T) {
    // --snip--
}
```

 trait bound  `?Sized` 意味着类型 `T` 可能是`Sized`也可能无法知道size。 问号修饰Trait的用法 `?Trait` 只能用在 `Sized`之前，不能用在其他trait之前. 