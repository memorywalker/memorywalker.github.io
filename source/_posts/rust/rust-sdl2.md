---
title: Rust SDL2 Develop
date: 2024-03-02 15:42:49
categories:
- programming
tags:
- rust
- learning
---

## RUST SDL2 Develop 

>  Rust Programming by Example . Chapter 2-3-4

### SDL2开发环境

#### 配置SDL

SDL2的官方https://www.libsdl.org/下载最新库文件 https://github.com/libsdl-org/SDL/releases/tag/release-2.30.0

对于windows下载[SDL2-devel-2.30.0-VC.zip](https://github.com/libsdl-org/SDL/releases/download/release-2.30.0/SDL2-devel-2.30.0-VC.zip)

SDL2库是由C语言实现的跨平台库，为了能在rust使用可以使用https://github.com/Rust-SDL2/rust-sdl2. 这个rust对SDL2封装，就能直接使用rust语言来开发。

安装`Rust-SDL2 ` https://github.com/Rust-SDL2/rust-sdl2. 在页面有详细的不同平台安装流程，对于Window MSVC环境：

1. 把下载的SDL2-devel-2.30.0-VC.zip中`SDL2-2.30.0\lib\x64\`的所有文件拷贝到rustup的库目录中`.rustup\toolchains\stable-x86_64-pc-windows-msvc\lib\rustlib\x86_64-pc-windows-msvc\lib\`

2. 使用`cargo new rtetris`创建一个工程名为`rtetris`的应用程序工程

3. 工程的`Cargo.toml`文件中增加以下依赖代码

   ```toml
   [dependencies]
   sdl2 = "0.36"
   ```

4. 把`SDL2.dll`文件拷贝到rust开发工程的根目录（和`Cargo.toml`相同目录）

#### 语义化版本(semantic version)

Semantic Versioning的版本有三个部分`[major].[minor].[patch]`

**major**: 重大修改且有不兼容的API变化

**minor**:增加新的功能，但不会破坏版本兼容性

**patch**: 修改bug的小更改

#### SDL特性设置

要使用sdl2的特性扩展，需要修改toml文件，不再使用之前的依赖写法，而针对sdl2单独写使用哪些特性

```toml
[dependencies.sdl2]
version = "0.36"
default-features = false
features = ["image"]
```

### 简单窗口程序

以下代码是一个简单的窗口程序，可以用测试程序是否可以正常编译

```rust
extern crate sdl2;

use sdl2::pixels::Color;
use sdl2::event::Event;
use sdl2::keyboard::Keycode;
use sdl2::rect::Rect;
use sdl2::render::{Texture, TextureCreator};

use std::time::Duration;
use std::thread::sleep;

const TEXTURE_SIZE : u32 = 32;

fn main() {
    // 初始化sdl
    let sdl_context = sdl2::init().expect("SDL Init failed");
    // 获取视频系统
    let video_subsystem = sdl_context.video().expect("Couldn't get sdl video subsystem");
    // 获取窗口，并设置窗口的属性，整个屏幕居中，使用opengl渲染
    let window = video_subsystem.window("rust-sdl2 demo: Video", 800, 600)
                    .position_centered()
                    .opengl()
                    .build()
                    .expect("Failed to create window");
    // 获取窗口画布，支持垂直同步
    let mut canvas = window.into_canvas()
                    .target_texture()
                    .present_vsync()
                    .build()
                    .expect("Failed to convert window into canvas");
    // 获取画布的纹理创建者
    let texture_creator: TextureCreator<_> = canvas.texture_creator();
    // 创建一个正方形纹理
    let mut square_texture: Texture = texture_creator.create_texture_target(None, TEXTURE_SIZE, TEXTURE_SIZE)
                .expect("Failed to create a texture");
    // 使用画布绘制纹理
    canvas.with_texture_canvas(&mut square_texture, |texture| {
        texture.set_draw_color(Color::RGB(0, 255, 0));
        texture.clear(); // 填充背景色
    }).expect("Failed to color a texture");

    // 事件句柄
    let mut event_pump = sdl_context.event_pump().expect("Failed to get SDL event pump");

    'running: loop {
        // 事件处理循环
        for event in event_pump.poll_iter() {
            match event {
                Event::Quit { .. } | 
                Event::KeyDown { keycode: Some(Keycode::Escape), ..} => 
                {
                    break 'running // 如果收到esc或关闭，退出这个事件循环
                },
                _=> {}
            }
        }
        // 绘制窗口的背景色
        canvas.set_draw_color(Color::RGB(255, 0, 0));
        canvas.clear();
        // 把纹理拷贝到窗口中的指定位置
        canvas.copy(&square_texture, None, Rect::new(0, 0, TEXTURE_SIZE, TEXTURE_SIZE))
                    .expect("Failed to copy texture into window");
        // 更新窗口显示
        canvas.present();

        // 每1秒60帧执行这个循环，所以要没1/60秒就sleep一下
        sleep(Duration::new(0, 1_000_000_000u32/60));
    }
}
```

执行`cargo run`只会的程序如下

![sdl2_demo](../../uploads/rust/sdl2_demo.png)
![sdl2_demo](/uploads/rust/sdl2_demo.png)

### 外部资源使用

#### 图片资源

##### 配置SDL的Image扩展库

SDL的图片插件地址为https://github.com/libsdl-org/SDL_image

把下载的[SDL2_image-devel-2.8.2-VC.zip](https://github.com/libsdl-org/SDL_image/releases/download/release-2.8.2/SDL2_image-devel-2.8.2-VC.zip)和SDL库一样配置。把其中的x64目录中的所有库文件放在rustup的库目录，把动态库文件也在工程目录中放一份。

##### 图片加载代码

书中代码编译不过，参考https://github.com/Rust-SDL2/rust-sdl2/blob/master/examples/image-demo.rs例子调整引用和初始化

```rust
use sdl2::image::{LoadTexture, InitFlag};
// 初始化图像上下文
let _image_context = sdl2::image::init(InitFlag::PNG | InitFlag::JPG).expect("Failed to initialize the image context");
// 创建一个图像纹理用来显示    
let image_texture = texture_creator.load_texture("res/images/flower.jpeg").expect("Failed to load image");
...
// 把图像纹理拷贝到窗口中
canvas.copy(&image_texture, None, None).expect("Failed to copy image to window");
```

其中图片资源放在工程根目录的`/res/images/`目录下

#### 读写文件

新建一个score_file.rs文件用来存取分数和行数。迭代器的next()在collect()调用的时候才会被执行。

```rust
use std::fs::File;
use std::io::{self, Read, Write};

fn write_into_file(content: &str, file_name: &str) -> io::Result<()> {
    let mut f = File::create(file_name)?;
    f.write_all(content.as_bytes())
}

fn read_from_file(file_name: &str) -> io::Result<String> {
    let mut f = File::open(file_name)?;
    let mut content = String::new();
    f.read_to_string(&mut content)?;
    Ok(content)
}

// 把数组中的每一个值转换为string类型，最后再把Vec<String>的每一个string用空格连接起来
fn slice_to_string(slice: &[u32]) -> String {
    slice.iter().map(|highscores| highscores.to_string())
                        .collect::<Vec<String>>().join(" ")
}
// 文件有两行，第一行存储分数列表，第二行存储函数列表
pub fn save_highscores_and_lines(highscores: &[u32], number_of_lines: &[u32]) -> bool {
    let s_highscores = slice_to_string(highscores);
    let s_num_of_lines = slice_to_string(number_of_lines);
    write_into_file(format!("{}\n{}\n", s_highscores, s_num_of_lines).as_str(),"save.txt").is_ok()
}

// 把一行文本中的字符用空格分割，并将每一个字串转换为u32类型的数字，最后返回一个vec
fn line_to_slice(line: &str) -> Vec<u32> {
    line.split(" ").filter_map(
        |nb| nb.parse::<u32>().ok())
        .collect()
}

// 分别读取两行文本，并把每一行的文本解析成数字的vec
pub fn load_highscores_and_lines() -> Option<(Vec<u32>, Vec<u32>)> {
    if let Ok(constent) = read_from_file("save.txt") {
        let mut lines = constent.splitn(2, "\n").map(
            |line| line_to_slice(line)).collect::<Vec<_>>();
        if lines.len() == 2 {
            let (number_lines, highscores) = (lines.pop().unwrap(), lines.pop().unwrap());
            Some((highscores, number_lines))
        } else {
            None
        }
    } else {
        None
    }
}
```

在main.rs文件中

```rust
mod score_file;

fn main() {
    let scores:[u32; 2] = [10, 20];
    let lines: [u32; 2] = [500,600];
    score_file::save_highscores_and_lines(&scores, &lines);
    if let Some(values) = score_file::load_highscores_and_lines() {
        println!("scores:{:?}, lines:{:?}", values.0, values.1); // scores:[10, 20], lines:[500]
    } else {
        println!("None data");
    }
}
```





