---
title: Rust Learning-Patterns and Matching
date: 2024-02-18 10:42:49
categories:
- rust
tags:
- rust
- learning
---

## RUST Patterns and Matching

[Rust 程序设计语言 - Rust 程序设计语言 简体中文版 (kaisery.github.io)](https://kaisery.github.io/trpl-zh-cn/title-page.html)

### 模式

Pattern是一种语法，用来匹配类型中的结构，一般和match配合使用。模式有点像正则表达式，**它检测一个值是否满足某种指定的规则，并从结构体或元组中一次性提取其成员到到本地变量中**，模式由以下几种类型组成：

1. Literals 字面值，写死的字串或数字
2. 结构的数组，枚举，结构体或元组
3. 变量
4. 通配符
5. 占位符

rust的表达式输出值，pattern消费值，**模式匹配可以把值分离成多个变量，而不是把值存储在一个变量中**。

#### 模式使用场景

##### match分支

match表达式所有可能值都必须被处理。一种确保处理所有情况的方法是在最后一个分支使用可以匹配所有情况的模式，如使用`_`模式匹配所有情况。

match表达式中`=>`左边的部分就是pattern，从上到下依次用VALUE与PATTERN进行匹配检测，如果匹配就执行右侧的表达式。

```rust
match VALUE {
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
}
```

例如下面的rt值为`RoughTime::InTheFuture(TimeUnit::Months, 1)`，对于第一个分支的模式，从左向右开始用值与之匹配检测，值的枚举为`InTheFuture`显然与分支的`InThePast`不匹配，因此用下一个分支检测，直到最后一个分支`RoughTime::InTheFuture(units, count)`，从左往右所有的数据类型都匹配，数值中与pattern匹配的值会被move或copy到pattern中的局部变量中，这里`TimeUnit::Months`赋值拷贝给了pattern中的局部变量`units`， 数值中1对应的赋值给了pattern中的变量`count`，在`=>`右侧的表达式中可以使用这两个局部变量的值。

```rust
fn rough_time_to_english(rt: RoughTime) -> String {
    match rt {
        RoughTime::InThePast(units, count) => {
            format!("{count} {} ago", units.plural())
        }
        RoughTime::JustNow => "just now".to_string(),
        RoughTime::InTheFuture(units, count) => {
            format!("{count} {} from now", units.plural())
        }
    }
}
```

##### if let条件

if let用来处理简单匹配一种情况的场景，当然也可以使用else来处理其他情况。if let, else if, else if let的条件可以是不相关的。编译器不会对if let的所有情况是否都覆盖了进行检查。if let可以和match一样使用覆盖变量 `shadowed variables` ，例如 `if let Ok(age) = age` 引入了一个新的shadowed `age` 变量，它包含了Ok变量中的值，它的作用域从if let的大括号的范围开始，所以`age > 30`中的age只能在if let代码块的内部有效。

```rust

if let Pattern = Expression {
    // 当Expression匹配Pattern时执行这里的代码
}

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

函数参数和let语句类似，形参变量就是模式，下面的实参 `&(3, 5)` 匹配模式 `&(x, y)` 从而把一个point变量分解成两个变量。

```rust
fn print_coordinates(&(x, y): &(i32, i32)) {
    println!("Current location: ({}, {})", x, y);
}

fn main() {
    let point = (3, 5);
    print_coordinates(&point);
}
```

##### 闭包参数

下面的例子中迭代器`iter()`返回的是元素的引用，使用`&num`模式可以解引用取得值后直接用于计算。

```rust
let numbers = vec![1, 2, 3, 4, 5];
let sum = numbers.iter().fold(0, |a, &num| a + num); // 15
```

迭代器类型的`fold`方法用来计算累计和。它有两个参数，参数1是累计的初始值，这里为0，只会调用一次；参数2是一个有两个参数的闭包，闭包的第一个参数是累计值，第二个参数为每个元素值(不是引用)，闭包的返回值为下一次迭代的累计值a。闭包会循环调用在每一个元素值上，从而计算出累计值。例如参数1如果为10，计算出的累计值为10+15=25。


##### 模式匹配的可反驳性

模式有两种形式 **refutable**可反驳的和**irrefutable**不可反驳的 。

不会出现匹配失败，可以匹配所有可能值的模式为不可反驳的，例如` let x = 5 `中x可以匹配所有值不会匹配失败

可能匹配失败的模式为可反驳的，例如 `if let Some(x) = a_value` ，如果值为None，Some(x)模式就会匹配失败。

函数参数、let语句、for循环、闭包只能接受不可反驳的模式，因为他们不能处理模式匹配失败的情况。对于if let、while let表达式可以接受不可反驳模式和可反驳模式，但是对于不可反驳模式由于模式不会失败，没有实际意义，所以编译器会提示编译警告。

### 模式语法

#### 字面值Literals

模式可以直接匹配字面值如数字1，字符，boolean，字符串等，主要用于比较和match表达式。这时的match和C中的switch语句类似。
下面的最后一个分支`n`匹配所有的整数。

```rust
let count = 10;
match count {
	0 => {} // nothing to say
	1 => println!("A rabbit is nosing around in the clover."),
	n => println!("There are {n} rabbits hopping about in the meadow"), // n is count
}
```
最后一个分支模式n可以起任何变量名字，在不同的情况下，它能匹配任何类型的值，例如下面的other就匹配了所有字串值。特殊的通配符`_`也可以看作一个本地变量，因此它能匹配任何值，只是rust不会把值拷贝给它，对于最后一个分支不需要使用值的情况，就可以使用`_`。

```rust
let month = "Oct";
let calendar = match month {
	"Jan" => String::from("January"),
	"Feb" => String::from("February"),
	"May" => String::from("May"),
	other => format!("other {:?}", other),
};

println!("calendar: {}", calendar); // calendar: other "Oct"
```
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

let at_end = match chars.peek() {
	Some('\r' | '\n') | None => true, // 字符为这三个情况都标识结束
	_ => false,
};
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

#### 匹配守卫(Match分支的额外条件保护)

可以在match分支的`模式`和`=>`之间再增加一个if语句进行进一步的条件判断 

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

结构体模式使用花括号表示，模式匹配时会对花括号中的每一个成员依次匹配

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
        Point { x: 0, y } => println!("On the y axis at {y}"), // this matched
        Point { x, y } => {
            println!("On neither axis: ({x}, {y})");
        }
    }
}
```

这个例子中第一个分支，匹配了所有y的值为0的结构体，第二个分支匹配了所有x的值为0的结构体。如果变量p的值定义为为`let p = Point { x: 0, y: 0 }`时，会执行第一个分支，因为match从第一个分支开始匹配，只要有一个匹配上，就不再执行了。

最后一个分支`Point { x, y }` 是结构体模式的简化写法，也可以写作`Point { x: x, y: y }`，rust会提示`^^^^ help: use shorthand field pattern: x`，建议使用简化写法。

当结构体的成员太多时，如果不需要使用其他成员的值，可以使用`..`代替其他成员，不用都列举出来。

```rust
match p {
	Point { x, y: 0 } => println!("On the x axis at {x}"),
	Point { x: 5, .. } => println!("Cross on x axis at 5"),
	Point { x, y } => {
		println!("On neither axis: ({x}, {y})");
	}
}
```

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

##### 元组

元组模式匹配元组数据，它主要用在一次操作多个数据的情况，例如下面的例子中同时处理了小时和上午或下午枚举。
```rust
/// Convert an hour AM or PM to the 24-hour convention.
/// For example, "4 P.M." is 16, and "12 A.M." is 0.
fn to_24_hour_time(hour: u32, half: DayHalf) -> u32 {
    match (hour, half) {
        (12, DayHalf::Am) => 0,
        (hour, DayHalf::Am) => hour,
        (12, DayHalf::Pm) => 12,
        (hour, DayHalf::Pm) => 12 + hour,
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

##### 数组和切片模式

当需要对一个数组的不同位置的数据做不同的处理时，可以对数组指定位置的元素进行模式匹配。例如HSL转换RGB颜色
```rust
fn hsl_to_rgb(hsl: [u8; 3]) -> [u8; 3] {
    match hsl {
        [_, _, 0] => [0, 0, 0], // 亮度为0时是黑色
        [_, _, 255] => [255, 255, 255], // 亮度为255时是白色
        _ => [0, 0, 0],
    }
}
```

切片不仅要匹配值还要匹配长度，切片模式只能和切片匹配，不能用于vec。

```rust
fn greet_people(names: &[String]) {
    match names {
        [] => println!("Hello, nobody."),
        [a] => println!("Hello, {a}."),
        [a, b] => println!("Hello, {a} and {b}."),
        [a, .., b] => println!("Hello, everyone from {a} to {b}."),
    }
}

greet_people(&[
        "Alice".to_string(),
        "Bob".to_string(),
        "Charlie".to_string(),
    ]); // Hello, everyone from Alice to Charlie.
```
 `greet_people`函数的参数names是指向一个切片的引用，所以模式中的变量a和b也是指向切片中对应元素的引用它们的类型为&String。
#### 使用@操作符把匹配值放入变量

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
    } // Found an id in range: 5
}
```

匹配切片中从某个位置开始的剩余元素

```rust
let ranked_teams = vec!["Alice", "Bob", "Charlie", "David", "Eve"];
let [first, second, others @ ..] = &ranked_teams[..] else {
    return;
};
assert_eq!(others, &ranked_teams[2..]); // ["Charlie", "David", "Eve"]
```
#### 引用匹配

匹配一个不可拷贝的值，会把这个值move进pattern的局部变量中，例如下面的例子中`cod`成员`name`已经被移动进局部变量`name`中，`Game`的其他成员已经被丢弃，所以后面的`output_game_info(&cod)`无法再继续使用这个变量值。
```rust
struct Game {
    id: u32,
    name: String,
    version: String,
}

fn find_game_by_name(name: &str) -> Option<Game> {
    None
}

fn output_game_info(game: &Game) {}

let cod = Game {
	id: 1,
	name: "Call of Duty".to_string(),
	version: "21".to_string(),
};

match cod {
	Game { id, name, version } => {
		println!("Game ID: {}", id);
		find_game_by_name(&name);
		output_game_info(&cod); // value borrowed here after partial move
	}
	_ => {}
}
```
这种情况下，可以匹配一个引用变量来把这个变量的引用传给模式的局部变量，由于现在匹配的是一个引用值，所以局部变量name也是引用对Game的name字段的引用，在传参的时候不需要`&`符号。

```rust
match &cod {
	Game { id, name, version } => {
		println!("Game ID: {}", id);
		find_game_by_name(name);
		output_game_info(&cod);
	}
	_ => {}
}
```

任何可以匹配类型`T`的地方都可以匹配`&T`或者`&mut T`. 在模式中不需要额外的标识，模式中的局部变量是对应匹配值的引用，而不会拷贝或move.例如上面的模式`Game { id, name, version }`的局部变量name就是cod的name值的引用。
一般情况下，在匹配的分支的中使用一个值的引用时，通常会像上面的例子匹配值的引用。
##### 借用模式

除了直接匹配一个值的引用，还可以使用借用模式borrowing pattern ，把匹配的值借用到模式的局部变量中。在模式变量前增加`ref` 或`ref mut`，从而不会拷贝或移动值。
```rust
    match cod {
        Game {
            id,
            ref name,
            ref version,
        } => {
            println!("Game ID: {}", id);
        }
    }
    println!("Game is {:?}", cod);
```
 Game结构体中有两个String类型的成员，它们都是不可拷贝的，所以想要它们不被移动到模式的局部变量中，必须两个成员前都加上ref标识借用对应的值的引用。
使用`ref mut`来借用一个可变引用
```rust
match line_result {
	Err(ref err) => log_error(err), // `err`是 &Error(shared ref)
	Ok(ref mut line) => {
		// `line`是 &mut String(mut ref)
		trim_comments(line);
		// 修改 String
		handle(line);
	}
}
```
##### 解引用模式

dereferencing pattern 使用`&`在模式变量的前面来匹配一个引用值，并解引用它。

```rust
match chars.peek() {
	Some(&c) => println!("coming up: {c:?}"),
	None => println!("end of chars"),
}
```
`chars`是一个字串的字符迭代器，它的`peek()`方法返回`Option<&char>`指向下一个字符的引用，这里可以使用`&c`获取这个字符，而不是字符的引用。

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

### 二叉树举例

```rust
// T类型的树结构.
pub enum BinaryTree<T> {
    Empty,
    NonEmpty(Box<TreeNode<T>>),
}

// 一个树的节点.
pub struct TreeNode<T> {
    element: T, // 当前节点的值
    left: BinaryTree<T>,
    right: BinaryTree<T>,
}

/// T的类型必须实现了Ord Trait，即可以比较大小
impl<T: Ord + std::fmt::Display> BinaryTree<T> {
    pub fn add(&mut self, value: T) {
        let mut place = self; // 临时变量缓存新节点位置
        while let BinaryTree::NonEmpty(node) = place { // 当前树不为空，即它有子节点
            if value <= node.element { // 新添加的值小于当前节点的值
                place = &mut node.left; // 新添加节点放在当前节点的左子树
            } else {
                place = &mut node.right;
            }
        }
        // 直到找到一个树为空，新的数据放在这个空位置上
        *place = BinaryTree::NonEmpty(Box::new(TreeNode {
            element: value,
            left: BinaryTree::Empty,
            right: BinaryTree::Empty,
        }));
    }
    /// 递归遍历
    pub fn traverse_in_order(&self) {
        match self {
            BinaryTree::Empty => {}
            BinaryTree::NonEmpty(node) => {
                node.left.traverse_in_order();
                println!("{}", node.element);
                node.right.traverse_in_order();
            }
        }
    }
}

fn main() {
    let mut tree = BinaryTree::Empty;
    tree.add("Mercury");
    tree.add("Venus");    
    tree.add("Earth");
    tree.add("Mars");
    tree.traverse_in_order(); \\ Earth Mars Mercury Venus
}
```