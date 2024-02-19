---
title: Rust Learning-Functional
date: 2024-01-01 11:42:49
categories:
- programming
tags:
- rust
- learning
---

## RUST Functional 

[Rust 程序设计语言 - Rust 程序设计语言 简体中文版 (kaisery.github.io)](https://kaisery.github.io/trpl-zh-cn/title-page.html)

### Functional Programming

函数作为一个对象，可以作为参数，返回值，给变量赋值然后执行

### Closure

闭包是一个匿名函数，他可以被存储在一个变量或作为另一个函数的参数。可以在一个地方定义闭包，然后再其他地方执行他，与函数不同的是，闭包在执行时可以获取他定义时所在上下文的值。

由于闭包没有名字且一般都是在很小的上下文范围内使用，编译器一般可以推断出闭包的参数类型和返回值类型

#### 基本写法

和普通函数写法类似，使用`||`传递参数，之后用大括号里面为函数体，当只有一句时，可以省略大括号。

如果一个闭包没有被使用，编译器无法推断出其数据类型，这个时候就不能省略其参数类型和返回值类型。编译器只会给闭包的参数和返回值推断一种数据类型，不能像模板一样支持多个类型。

```rust
let add_one_1 = | x: u32| -> u32 { x + 1 };
let add_one_2 = | x | { x + 1 };
let add_one_3 = | x |  x + 1 ;

let mut num = 0;
num = add_one_1(num);
num = add_one_2(num);
num = add_one_3(num);

let mut fnum = 0.0;
fnum = add_one_2(fnum); // 闭包的数据类型在前面已经被推导为u32了，这里会编译错误

println!("final num:{num}"); // final num:3
```

#### 闭包使用外部值

在闭包中使用外部值分为三种情况（和函数参数相同）：

* 作为不可变引用immutable reference
* 作为可变引用 mutable reference
* 获取所有权 taking ownership，在 `||`前使用`move`


编译器会根据场景使用最小的使用权。

```rust
use std::thread;
fn main() {

    let mut list = vec![1, 2, 3];
    println!("Before defining closure: {:?}", list);
    // 因为print只需要不可变引用，所以这里list只是不可变引用
    let only_borrows = || println!("From closure: {:?}", list);

    println!("Before calling closure: {:?}", list);
    only_borrows();
    println!("After calling closure: {:?}", list);

    // 在此之前list只是被不可变引用，但这里会修改list，此时会变化为可变引用
    let mut borrows_mutably = || list.push(7);
    // 闭包还没结束，所以list还是可变引用，这时不能作为不可变引用使用
    // println!("before calling closure: {:?}", list); // error
    borrows_mutably();
    println!("After calling closure: {:?}", list); // 闭包结束，又可以按不可变引用使用

    // 子线程中要使用list值，但是主线程main可能已经执行完了，导致list值被释放，所以要把list的所有权转移到子线程中
    thread::spawn(move || println!("From thread: {:?}", list))
        .join()
        .unwrap();
    // list已经被move到子线程中，main线程不能再使用了
    //println!("After calling thread: {:?}", list); // error
}
```

#### Fn Traits

闭包体内如何对外部引用使用方法决定了闭包实现类哪种类型的`Fn trait`，而函数或结构体可以指定自己使用哪种`Fn trait`类型的闭包。闭包会**自动**实现三种类型的`Fn trait`，这三种类型从严格到宽松。

* `FnOnce `这种闭包只能被执行一次。所有的闭包都实现了这个trait。当一个闭包把一个获取的引用移出了闭包体，这个闭包只能是`FnOnce `。
* `FnMut` 这种闭包不会把引用值移出闭包体，但是会修改引用的值。这种闭包可以被调用多次。
* `Fn `这种闭包不会改变引用值，就像没有从外部获取值一样。这种闭包可以被调用多次，即使在多线程时调用也不影响。

当然普通的函数也可以实现以上三种`Fn traits`。

`Option<T>`的`unwrap_or_else`方法声明了它会使用`FnOnce`的闭包，当`impl<T> `的值为None时，它会调用传入的闭包`f`一次，这个闭包返回的类型为T。

```rust
impl<T> Option<T> {
    pub fn unwrap_or_else<F>(self, f: F) -> T
    where
        F: FnOnce() -> T
    {
        match self {
            Some(x) => x,
            None => f(),
        }
    }
}
```

例如以下例子中，list的get返回一个Option<&Rectangle>，如果值为None时，使用闭包输出不存在，并返回一个新的Rectangle对象。

```rust
let mut list = [
  Rectangle { width: 10, height: 1 },
  Rectangle { width: 3, height: 5 },
  Rectangle { width: 7, height: 12 },
];

// FnOnce
let rect = list.get(3).unwrap_or_else(|| {
  println!("cant find"); 
  &Rectangle { width: 0, height: 0 }
});

println!("{:#?}", rect);
```

对list排序的`sort_by_key`方法，就使用`FnMut`类型的闭包，因为这个闭包里面把list的一个元素作为参数传入，返回一个可以用作排序的值K。例如使用长方形的款作为排序的key，其中闭包获取一个元素r作为入参，返回r的宽度作为排序比较的key值，虽然这个方法使用的闭包不会修改任何值，但是他需要这个闭包可以被多次执行以遍历list中的所有元素，所以它使用的闭包类型定义为`FnMut`.

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let mut list = [
        Rectangle { width: 10, height: 1 },
        Rectangle { width: 3, height: 5 },
        Rectangle { width: 7, height: 12 },
    ];

    // FnMut
    list.sort_by_key(|r| r.width);
    println!("{:#?}", list);
}
```

对于只实现了`FnTrait`的闭包`sort_by_key`就不能使用。闭包体中的`sort_operations.push(value)`从外部获取value的所有权，并将所有权又传出去给了外部变量`sort_operations`，导致**下一次**执行这个闭包时，已经无法获取到value的所有权了。而`num_sort_operations`变量只是可变引用，可以被多次执行。

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let mut list = [
        Rectangle { width: 10, height: 1 },
        Rectangle { width: 3, height: 5 },
        Rectangle { width: 7, height: 12 },
    ];

    let mut sort_operations = vec![];
    let value = String::from("by key called");

    let mut num_sort_operations = 0;

    list.sort_by_key(|r| {
        // cannot move out of `value`, a captured variable in an `FnMut` closure
        sort_operations.push(value); // error
        num_sort_operations += 1; // ok
        r.width
    });
    println!("{:#?}", list);
}
```

#### 返回闭包

闭包可以看做是一种trait，所以不能直接返回它，因为trait的大小是未知的。但是可以通过trait object方式返回闭包。即给闭包增加一个指针。

```rust
fn returns_closure() -> Box<dyn Fn(i32) -> i32> {
    Box::new(|x| x + 1)
}
```

### 函数指针

函数也可以作为参数传递给另一个函数。`fn`类型称作函数指针。使用函数指针可以复用已经实现过的函数。

例如已经有了一个实现整数加1的函数，我们想实现整数加一操作执行多次，就可以在新的函数中调用已经实现的函数。

```rust
fn add_one(x: i32) -> i32 {
    x + 1
}

fn do_repeat(f: fn(i32) -> i32, arg: i32, time: i32) -> i32 {
    let mut val = 0;
    for _i in 0..time {
        val = val + f(arg); // 调用函数指针
    }
    val
}

fn main() {
    let answer = do_repeat(add_one, 2, 5);
    println!("The answer is: {}", answer);// The answer is: 15
}
```

函数指针实现了三种类型的闭包，所以可以使用闭包的地方，都可以使用函数指针。

```rust
let list_of_numbers = vec![1, 2, 3];
let list_of_strings: Vec<String> =
    list_of_numbers.iter().map(|i| i.to_string()).collect();// 使用闭包
let list_of_strings: Vec<String> =
    list_of_numbers.iter().map(ToString::to_string).collect(); // 使用函数指针
```

枚举的每个变量名也是一个初始化函数，所以这个变量名也是函数指针。

```rust
enum Status {
    Value(u32),
    Stop,
}
// 使用Status::Value(u32)来对每一个从0到20的u32类型的数值创建Status::Value实例
let list_of_statuses: Vec<Status> = (0u32..20).map(Status::Value).collect(); 
```


### 迭代器Iterator

迭代器模式可以对一系列数据元素逐个访问。迭代器对象是懒加载的，只有消费了迭代器，它才会执行遍历。

#### 迭代器Trait

`Iterator` trait有一个`next()`方法，它返回一个`Option<Self::Item>`类型对象，其中的Item是这个迭代器的关联类型。当迭代器遍历完所有元素后，next返回None.

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    // methods with default implementations elided
}

let mut v1 = vec![1, 2, 3];

let mut immut_iter = v1.iter();
assert_eq!(immut_iter.next(), Some(&1));

let mut mut_iter = v1.iter_mut();

let mut owner_iter = v1.into_iter();
```

一般定义的迭代器类型都是`mut`类型因为执行`next`方法会修改迭代器对象内的索引。

* 使用`iter()`获取到原始列表的v1不可变引用
* 使用`iter_mut()`获取到原始列表的v1可变引用
* 使用`into_iter()`获取到原始列表v1的所有权

#### 消费迭代器

`Iterator` trait中定义了一些方法调用next方法称为`consuming adaptors`，因为他们通过next遍历每一个元素从而用尽迭代器。例如`sum()`方法就遍历所有元素累加各个元素的和，同时它会获取迭代器的所有权。

```rust
fn iterator_sum() {
    let v1 = vec![1, 2, 3];

    let v1_iter = v1.iter();
    let total: i32 = v1_iter.sum();
    assert_eq!(total, 6);

    let total: i32 = v1_iter.sum(); // error, use of moved value: `v1_iter`
}
```

#### 生产迭代器

有些 `Iterator` trait的方法可以以迭代器作为输入并产生变化后的迭代器，这些方法称作*Iterator adaptors* 。例如map()会对迭代器的每一个元素执行指定的闭包操作，并返回一个新的迭代器。由于这个新的迭代器是懒加载，所以需要执行collect()使其转换为一个vector。

```rust
let v1: Vec<i32> = vec![1, 2, 3];
let v : Vec<_>= v1.iter().map(|x| x + 1).collect();
```

#### 使用闭包和迭代器

 `filter` 方法使用一个闭包作为参数，遍历每一个元素过程中，当闭包返回true时，就把这个元素加新生成的迭代器中，如果返回false，就丢掉。

```rust
#[derive(PartialEq, Debug)]
struct Shoe {
    size: u32,
    style: String,
}

// 第一个参数获取了shoes的所有权，并返回了一个新的Vec<Shoe>
fn shoes_in_size(shoes: Vec<Shoe>, shoe_size: u32) -> Vec<Shoe> {
    shoes.into_iter().filter(|s| s.size == shoe_size).collect()
}

fn main() {
    let shoes = vec![
        Shoe {
            size: 10,
            style: String::from("sneaker"),
        },
        Shoe {
            size: 13,
            style: String::from("sandal"),
        },
        Shoe {
            size: 10,
            style: String::from("boot"),
        },
    ];
	// 过滤大小为10的shoes，shoes的所有权被转移走，后续不能再使用
    let in_my_size_shoes = shoes_in_size(shoes, 10);
	// 新返回的shoes的vector
    println!("{:#?}",in_my_size_shoes);
}
```

### 迭代器性能

使用迭代器虽然看似高层次的抽象，但是rust编译器最终会对代码优化，不会带来额外的运行成本，甚至可能比直接手写for循环效率高。迭代器是rust中零成本**zero-cost**抽象的一个特性。

Bjarne Stroustrup, the original designer and implementor of C++, defines *zero-overhead* in “Foundations of C++” (2012):

> In general, C++ implementations obey the zero-overhead principle: What you don’t use, you don’t pay for. And further: What you do use, you couldn’t hand code any better.

书中举了一个音频编码的例子，

```rust
let buffer: &mut [i32];
let coefficients: [i64; 12];
let qlp_shift: i16;

for i in 12..buffer.len() {
    let prediction = coefficients.iter()
                                 .zip(&buffer[i - 12..i])
                                 .map(|(&c, &s)| c * s as i64)
                                 .sum::<i64>() >> qlp_shift;
    let delta = buffer[i];
    buffer[i] = prediction as i32 + delta;
}
```

对于coefficients遍历，rust知道其中有12个元素，为了减少循环控制代码性能损耗，rust会生成12个重复的代码来优化这个循环。

Rust knows that there are 12 iterations, so it “unrolls” the loop. *Unrolling* is an optimization that removes the overhead of the loop controlling code and instead generates repetitive code for each iteration of the loop.




