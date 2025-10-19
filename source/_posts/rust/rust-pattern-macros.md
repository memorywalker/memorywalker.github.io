---
title: Programming Rus - Macros
date: 2025-10-18 15:42:49
categories:
- programming
tags:
- rust
---

## RUST Macros

### 宏

宏在程序代码编译为机器码之前会被展开为rust代码，所以它与函数调用不同。rust中的宏和c++中的宏类似，但是rust的宏有语法检查，不像C++的宏只是纯粹的文本展开。



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

