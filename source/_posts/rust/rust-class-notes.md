---
title: Rust 第一课笔记
date: 2026-01-31T10:34:00
categories:
  - rust
tags:
  - rust
  - network
---
## Rust 第一课笔记

原文地址：
https://learn.lianglianglee.com/%e4%b8%93%e6%a0%8f/%e9%99%88%e5%a4%a9%20%c2%b7%20Rust%20%e7%bc%96%e7%a8%8b%e7%ac%ac%e4%b8%80%e8%af%be/20%204%20Steps%20%ef%bc%9a%e5%a6%82%e4%bd%95%e6%9b%b4%e5%a5%bd%e5%9c%b0%e9%98%85%e8%af%bbRust%e6%ba%90%e7%a0%81%ef%bc%9f.md

### 写作与Coding

写代码和写作文类似，从小通过学基本词汇，语法达到能写的基本要求，通过读名家著作学习写作技巧，在掌握了一种语言的基本语法和关键词后，可以通过阅读经典源代码来提高自己的写作水平。

通过阅读也就知道一个事情可以有不同的描写方法，一个需求也可以有多个不同的实现方案，通过广泛阅读，扩展自己的技能或技巧，以后自己写作的时候也可以使用不同的修辞，文章结构构思。

### Rust代码阅读

1. 在docs.rs或者lib.rs中注释以及readme了解这个库的基本功能，有哪些接口和结构
2. 查看关键trait(接口)信息，例如:
    1. 要求必须实现哪些方法(**Required Methods**)
    2. 能提供哪些方法(**Provided Methods**)
    3. (**Implementations on Foreign Types**)它为哪些外部类型已经实现这个trait，例如Bytes库中已经为切片 `&[u8]`、`VecDeque` 都实现了 `Buf trait`
    4. (**Implementors**) 这个Crate中哪些结构实现了这个trait
3. 熟悉主要的结构体struct：
    1. 结构体的内存布局，整体结构
    2. 实现了哪些trait，有哪些方法Methods
4. 看库的example或test熟悉这个库的基本用法
5. **主题阅读**对于自己感兴趣主题，深入学习其中的实现，总结出实现细节，为自己以后实现类似功能做积累

**经验**：

- 定义好 trait 后，**可以考虑一下标准库的数据结构**，哪些可以实现这个 trait。
- 如果未来别人的某个类型 T ，实现了你的 trait，**那他的 &T、&mut T、Box 等衍生类型，是否能够自动实现这个 trait**。
- **我们自己的数据结构，也应该尽可能实现需要的标准 trait**，包括但不限于：AsRef、Borrow、Clone、Debug、Default、Deref、Drop、PartialEq/Eq、From、Hash、IntoIterator（如果是个集合类型）、PartialOrd/Ord 等。