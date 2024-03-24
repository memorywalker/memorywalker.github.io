---
title: Rust Tips
date: 2024-03-24 16:25:49
categories:
- programming
tags:
- rust
- learning
---

## Rust Tips 



### 基本用法

#### 字节流转自定义数据类型

从一个二进制文件中读取一个结构

rust标准库内部使用mem来把4字节数据转换为float类型，反之亦然 https://doc.rust-lang.org/src/core/num/f32.rs.html

```rust
pub const fn from_bits(v: u32) -> Self {       
        const fn ct_u32_to_f32(ct: u32) -> f32 {
            match f32::classify_bits(ct) {
                FpCategory::Subnormal => {
                    panic!("const-eval error: cannot use f32::from_bits on a subnormal number")
                }
                FpCategory::Nan => {
                    panic!("const-eval error: cannot use f32::from_bits on NaN")
                }
                FpCategory::Infinite | FpCategory::Normal | FpCategory::Zero => {
                    // SAFETY: It's not a frumious number
                    unsafe { mem::transmute::<u32, f32>(ct) }
                }
            }
        }       
    }
   
    pub const fn to_bits(self) -> u32 {       
        const fn ct_f32_to_u32(ct: f32) -> u32 {
            match ct.classify() {
                FpCategory::Nan => {
                    panic!("const-eval error: cannot use f32::to_bits on a NaN")
                }
                FpCategory::Subnormal => {
                    panic!("const-eval error: cannot use f32::to_bits on a subnormal number")
                }
                FpCategory::Infinite | FpCategory::Normal | FpCategory::Zero => {
                    // SAFETY: We have a normal floating point number. Now we transmute, i.e. do a bitcopy.
                    unsafe { mem::transmute::<f32, u32>(ct) }
                }
            }
        }        
    }
```

解析结构体可以使用标准库的方法，也可以使用第三方的crate byteorder，甚至可以自己直接使用unsafe来解析字节数据

```rust
#[derive(Debug)]
struct Header {
    pub magic: u16,
    pub version: u8,
    pub size: u32,
    pub ratio: f32,
}

impl Header {
    fn from(data: &[u8]) -> Header {
        Header {
            magic: u16::from_le_bytes(data[0..2].try_into().unwrap()),
            version: data[2],
            size: u32::from_le_bytes(data[3..7].try_into().unwrap()),
            ratio: f32::from_le_bytes(data[7..11].try_into().unwrap()),
        }
    }
} 

fn test_bin() {
    let pi:f32 = 3.14159265358979323846;
    let mut fdata = pi.to_le_bytes();
    let mut data = vec![0x04, 0x00, 0x01, 0x02, 0x00, 0x00, 0x00];
    data.extend_from_slice(&mut fdata);
    let header = Header::from(&data);
    println!("The result is {:?}", header);
}
```

