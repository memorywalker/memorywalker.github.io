---
title: Rust Learning-Collections
date: 2023-12-31 14:42:49
categories:
- programming
tags:
- rust
- learning
---

## RUST Collections 

[Rust 程序设计语言 - Rust 程序设计语言 简体中文版 (kaisery.github.io)](https://kaisery.github.io/trpl-zh-cn/title-page.html)

### Collection

容器的数据存储在堆上，在运行时可以改变大小

### Vector

`Vec<T>`使用泛型实现了列表容器，其元素顺序存储且数据类型必须相同。

#### 基本操作

* 使用`Vec::new()`创建一个vector对象，后续再给他添加值
* 使用`vec!`宏根据初始化数据创建一个vector对象
* 使用`push`添加元素
* 使用下标索引[index]获取元素
* 使用get函数获取`Option<&T>`，获取的Option可以用来判断是否越界访问，例如只有3个元素的vector，使用`get(3)`，就会返回None

```rust
let values : Vec<i32> = Vec::new(); // 需要使用类型注解Vec<i32>，告诉编译器类型
let mut lines = vec![1, 2, 3]; // 编译器根据数据推导出类型

let mut num  = Vec::new();  // 编译器根据下面的push，知道数据类型为i32
num.push(3);
num.push(2);

let first = lines[0];  // copy
//let first = &lines[0];   // 第一个元素被不可变引用，后面push修改vector需要一个可变引用，开始第一个元素内存区域
let second = &lines[1];  // 不知道为什么不会报错，只有第一个会报错
let third = lines.get(2);  // get Option<&T> back
//lines.push(5); // cannot borrow `lines` as mutable because it is also borrowed as immutable

match third {
  	Some(third) => println!("The third element is {third}"),
  	None => println!("There is no third element."),
}

println!("vec {:?} and first {first}", lines);
```

当对vector的第一个元素使用了不可变引用后，再对vector执行push方法，会提示vector已经被不可变引用了，不能再以可变引用的方式使用了。因为vector的元素连续存储，如果添加一个元素导致vector重新申请内存调整位置，之前引用的内存区域就会被释放了。

#### 遍历Vector

使用for遍历一个vector其中的元素可以可变或不可变两种方式引用，因此不能在循环中修改vector的大小

```rust
let mut v = vec![100, 32, 57];   

for i in &mut v { // mutable references
  *i += 50; // 使用* 解引用获取数值
}

for i in &v {
  println!("{i}");
  //v.push(55); // error
}
v.push(55);    // ok 循环变量不再引用vector了
```

#### enum元素

由于vector要求其中的元素类型必须相同，但可以使用枚举的方式扩展这种限制，因为同一个枚举中的变体可以有不同的类型。但是这种方法要求编译期就知道vector中元素的种类的占用的内存大小。

```rust
#[derive(Debug)]
enum SpreadsheetCell {
  Int(i32),
  Float(f64),
  Text(String),
}

let mut row = vec![
  SpreadsheetCell::Int(3),
  SpreadsheetCell::Text(String::from("blue")),
  SpreadsheetCell::Float(10.12),
]; 

row[0] = SpreadsheetCell::Text(String::from("black")); // 编译器知道需要多少空间

for item in  row{
  println!("{:?}", item);
}
// 输出：
Text("black")
Text("blue")
Float(10.12)
```

### String

`String`类型在rust标准库中定义是一个可扩大、可变、可拥有的UTF8编码的字串类型。

`str`类型是在rust核心库中定义，用来表示字符串slice的utf8编码的字串，一般以引用`&str`的方式使用

`String`使用vector来实现`Vec<u8>`

#### 基本操作

##### 创建

```rust
    let mut s = String::new();
    let data = "initial contents"; // data is &str type
    let s = data.to_string(); // s is String type    
    // the method also works on a literal directly:
    let s = "initial contents".to_string(); // s is String type
    let s = String::from("initial contents");
```

##### 修改

使用`push_str(&mut self, string: &str)`在字串后追加字串

使用`push(&mut self, ch: char)`在字串后追加字符

使用`+`操作符连接两个字符串，这个操作符本质上是`fn add(self, s: &str) -> String`，他的第二个参数是一个字串切片，第一个参数没有&符号，所以会获取+号前的对象的所有权，同时把拼接后的字串的所有权返回出来，由于没有拷贝，所以效率会高一些。

对于复杂的字符串拼接，可以使用`format!`宏，它不会获取变量的所有权

```rust
let mut s1 = String::from("foo");
let s2 = "bar";
s1.push_str(s2);
println!("s2 is {s2}");

let mut s = String::from("lo");
s.push('l');

let s1 = String::from("Hello, ");
let s2 = String::from("world!");
// Rust uses a deref coercion, which here turns &s2 into &s2[..]. 
// 编译器会强制把String类型转换为切片类型作为参数传给add
let s3 = s1 + &s2; // note s1 has been moved here and can no longer be used

let s1 = String::from("tic");
let s2 = String::from("tac");
let s3 = String::from("toe");
let s = format!("{s1}-{s2}-{s3}"); // format!不会转移所有权
// 多个字串拼接时，后两个是引用
let s = s1 + "-" + &s2 + "-" + &s3;
//let s = format!("{s1}-{s2}-{s3}"); // s1已经不能用了
```

##### 子串操作

rust的string不支持直接使用索引来获取一个字串中的字符，因为utf8字节流以vector的方式存储，取其中的一个字节出来一般不是预期的字符，为了避免预期外的错误，rust就不支持这种操作了。

可以使用区间操作获取一个字串切片`&str`类型，但是需要保证切割的字节数刚好满足字符边界

```rust
let hello = "Здравствуйте";

let s = &hello[0..3]; // 每个字符占两个字节，取前三个字节运行时会出错
println!("{s}"); 
// byte index 3 is not a char boundary; it is inside 'д' (bytes 2..4) of `Здравствуйте`
```

需要根据自己需要子串的数据类型选择合适的方法，例如要获取字符，使用`chars()`，如果想获取字节数据使用`bytes()`

```rust
let hello = "Здравствуйте";

for c in hello.chars() {
    print!("{c} "); // З д р а в с т в у й т е
}

for b in hello.bytes() {
    print!("{b} "); // 208 151 208 180 209 128 208 176 208 178 209 129 209 130 208 178 209 131 208 185 209 130 208 181 %
}
```

### Hash Map

`HashMap<K, V>`使用hash函数计算一个键值在内存中的位置。同一个map要求所有key的类型相同，所有值的类型相同。可以修改hash函数算法，默认使用的是*SipHash*

##### 基本操作

*   使用insert插入元素
*   使用get获取key对应的值，返回`Option<&V>`

```rust
use std::collections::HashMap; 

let mut scores = HashMap::new();
scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);

let name = String::from("Blue");
// 如果get返回为None，就返回默认值0
let score = scores.get(&name).copied().unwrap_or(0);
println!("socre={score}");

for (key, value) in scores {
    println!("{key}, {value}");
}
```

##### 修改Map

分三种情况:

1. 覆盖原有key对应的值
2. 判断如果key不存在就添加，已经存在不处理
3. 修改一个已经存在key的值

`HashMap`的`entry`方法以`key`为参数返回一个`Entry`类型的枚举，用来表示一个值是否存在。

```rust
let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);
scores.insert(String::from("Blue"), 20); // 1. 副高已经存在key的值

scores.entry(String::from("Black")).or_insert(50); // 2. 如果不存在才添加
scores.entry(String::from("Blue")).or_insert(50); // 如果已经存在，什么都不做

let text = "hello world wonderful world";
let mut map = HashMap::new();

for word in text.split_whitespace() {
    let count = map.entry(word).or_insert(0);
    *count += 1; // 3. 修改value的值，需要先解引用
}

println!("{:?}", map); // {"world": 2, "hello": 1, "wonderful": 1}\
```

