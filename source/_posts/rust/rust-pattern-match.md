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

#### 字面值Literals

模式可以直接匹配字面值如数字1，字符串等，主要用于比较和match表达式。

#### 匹配有名变量

```rust
fn main() {
    let x = Some(5);
    let y = 10;

    match x {
        Some(50) => println!("Got 50"),
        Some(y) => println!("Matched, y = {y}"),
        _ => println!("Default case, x = {:?}", x),
    }

    println!("at the end: x = {:?}, y = {y}", x);
}
//Matched, y = 5
//at the end: x = Some(5), y = 10
```

在match中，x作为值会依次和三个pattern匹配，x的值为5所以和第一个分支不匹配，第二个分支比较特殊，它在match的代码块中引入了一个新的变量y，这个y值会覆盖shadow外面定义的`y = 10`，这个y与任何在`Some`中的值匹配，所以它与`Some(5)`是匹配的，所以会执行第二个分支，并输出y的值为5。如果x的值为None，就会执行最后一个`_`分支，因为下划线匹配任何值。

当match表达式执行完成后，内部覆盖的y作用域结束，y的值又会是外部定义的y的值10。

#### 多重模式

多个模式可以使用`|`类似或一样组合起来，下面的例子中，无论x的值为1或2，都会走第一个分支

```rust
    let x = 2;

    match x {
        1 | 2 => println!("one or two"),
        3 => println!("three"),
        _ => println!("anything"),
    }
```

#### 匹配一个范围的模式

`start..=end`,标识start到end之间的所有值，包括end的值，只支持数字和字符类型。x的值为1-5的值时，都执行第一个分支。

```rust
    let x = 2;

    match x {
        1..=5 => println!("one through five"),
        _ => println!("something else"),
    }
```

#### Match的额外条件保护

可以在match的分支中再增加一个if语句进行进一步的条件判断 

```rust
fn main() {
    let num = Some(5);

    match num {
        Some(x) if x % 2 == 0 => println!("The number {} is even", x),
        Some(x) => println!("The number {} is odd", x),
        None => (),
    }
}
```

当num的值为4时，满足第一个分支，进而判断x是偶数，所以执行这个分支的表达式；当num的值为5时，虽然满足了match的第一个分支，但是后面的额外条件保护不满足，所以会继续判断match的第二个分支，从而输出第二个分支的表达式。

#### 使用模式解构枚举、结构体和元组

解构可以让我们方便使用结构体或元组中的一部分变量数据

##### 结构体

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 0, y: 7 };

    let Point { x: a, y: b } = p;
    assert_eq!(0, a);
    assert_eq!(7, b);
}
```

通过定义`Point { x: a, y: b }`结构体模式，来让a和b分别匹配解构体的两个成员x和y，也可以使用结构体成员本来的名字来作为匹配的变量。下面的例子中，直接就可以使用x和y作为模式匹配变量

```rust
fn main() {
    let p = Point { x: 0, y: 7 };

    let Point { x, y } = p;
    assert_eq!(0, x);
    assert_eq!(7, y);
}
```

还可以使用字面值作为匹配的变量

```rust
fn main() {
    let p = Point { x: 0, y: 7 };

    match p {
        Point { x, y: 0 } => println!("On the x axis at {x}"),
        Point { x: 0, y } => println!("On the y axis at {y}"),
        Point { x, y } => {
            println!("On neither axis: ({x}, {y})");
        }
    }
}
```

这个例子中第一个分支，匹配了所有y的值为0的结构体，第二个分支匹配了所有x的值为0的结构体。如果变量p的值定义为为`let p = Point { x: 0, y: 0 }`时，会执行第一个分支，因为match从第一个分支开始匹配，只要有一个匹配上，就不再执行了。

##### 枚举

枚举匹配和具体的元组，结构体匹配是相同的语法

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

fn main() {
    let msg = Message::ChangeColor(0, 160, 255);

    match msg {
        Message::Quit => {
            println!("The Quit variant has no data to destructure.");
        }
        Message::Move { x, y } => {
            println!("Move in the x direction {x} and in the y direction {y}");
        }
        Message::Write(text) => {
            println!("Text message: {text}");
        }
        Message::ChangeColor(r, g, b) => {
            println!("Change the color to red {r}, green {g}, and blue {b}",)
        }
    }
}
```

##### 嵌套的枚举、结构体和元组

在一个枚举中匹配另一个枚举

```rust
enum Color {
    Rgb(i32, i32, i32),
    Hsv(i32, i32, i32),
}

enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(Color),
}

fn main() {
    let msg = Message::ChangeColor(Color::Hsv(0, 160, 255));

    match msg {
        Message::ChangeColor(Color::Rgb(r, g, b)) => {
            println!("Change color to red {r}, green {g}, and blue {b}");
        }
        Message::ChangeColor(Color::Hsv(h, s, v)) => {
            println!("Change color to hue {h}, saturation {s}, value {v}")
        }
        _ => (),
    }
}
//Change color to hue 0, saturation 160, value 255
```

结构体嵌套在元组中

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let ((feet, inches), Point { x, y }) = ((3, 10), Point { x: 3, y: -10 });
    println!("feet {feet}, inches {inches}, x={x}, y={y}");
}// feet 3, inches 10, x=3, y=-10
```

##### 使用@操作符把匹配值放入变量

对于第一个分支，id的值5匹配了3-7之间，同时我们可以使用`id_variable @`来让`id_variable`变量保存匹配的值5。对于第二个分支，如果msg的值为10，即使匹配到了这个分支，但是由于没有变量保存匹配的值，所以无法知道具体匹配值是多少；第三个分支和普通的结构体模式相同，它匹配结构体的成员id，所以可以把id的值打印出来。

```rust
enum Message {
        Hello { id: i32 },
}

fn main() {
    let msg = Message::Hello { id: 5 };

    match msg {
        Message::Hello {
            id: id_variable @ 3..=7,
        } => println!("Found an id in range: {}", id_variable),
        Message::Hello { id: 10..=12 } => {
            println!("Found an id in another range")
        }
        Message::Hello { id } => println!("Found some other id: {}", id),
    }
}
```



#### 忽略模式中的值

##### 忽略所有值

```rust
fn foo(_: i32, y: i32) {
    println!("This code only uses the y parameter: {}", y);
}

fn main() {
    foo(3, 4);
}
```

使用`_`标识这个参数不在函数中被使用，例如接口发生变化后，如果不想修改函数签名，就可以把不用的参数设置为`_`，不会出现编译警告。这个方法在给一个结构体实现trait的方法时，如果这个结构体不会用trait的方法声明中的参数也可以用`_`代替。

```rust
trait Draw {
    fn draw(&self, w:i32, h:i32);
}

struct Square {
    side: i32,
}

impl Draw for Square {
    fn draw(&self, w:i32, _:i32) {
        println!("draw a square with {}", w);
    }
}
```

##### 忽略部分值

在模式中使用`_`可以忽略部分值

```rust
let numbers = (2, 4, 8, 16, 32);

match numbers {
    (first, _, third, _, fifth) => {
        println!("Some numbers: {first}, {third}, {fifth}")
    }
}// 元组中的4和16就会被忽略掉
```

下面的例子中，分支一不关心具体的值是多少，只要两个值都是Some就行，当两个值中有任何一个为None，就会执行第二个分支

```rust
    let mut setting_value = Some(5);
    let new_setting_value = Some(10);

    match (setting_value, new_setting_value) {
        (Some(_), Some(_)) => {
            println!("Can't overwrite an existing customized value");
        }
        _ => {
            setting_value = new_setting_value;
        }
    }

    println!("setting is {:?}", setting_value);

```

##### 忽略不使用的变量

变量名使用`_`开始可以告诉编译器这个变量不被使用，不用警告了，目前不知道有什么作用。编译器也会提示

`if this is intentional, prefix it with an underscore: `_y``

```rust
fn main() {
    let _x = 10;
    let y = 100;
    println!("unused value {}", _x);
}
```

名字有下划线前缀的变量和其他变量相同，if let语句中s会被移动到`_s`，所以后面在去打印s的值，会导致编译错误。

```rust
fn main() {
    let s = Some(String::from("Hello!"));
    //if let Some(_s) = s {// error borrow of partially moved value: `s`
    if let Some(_) = s {
        println!("found a string");
    }
    println!("{:?}", s);
}
```

##### 忽略剩余值

可以使用`..`标识结构体或元组的剩下的变量。例如结构体有很多成员，我们只想获取其中一个成员的值，其他的成员就可以用`..`代替

```rust
    struct Point {
        x: i32,
        y: i32,
        z: i32,
    }

    let origin = Point { x: 0, y: 0, z: 0 };

    match origin {
        Point { x, .. } => println!("x is {}", x),
    }
```

也可以用`..`代替一个区间的所有值剩余变量，编译器会判断`..`标识的变量是否存在歧义，例如下面的例子`..`就可以标识中间的所有值

```rust
fn main() {
    let numbers = (2, 4, 8, 16, 32);

    match numbers {
        (first, .., last) => {
            println!("Some numbers: {first}, {last}");
        }
    }
}//  Some numbers: 2, 32
```

