---
title: Rust实现最简单的Flappy Bird
date: 2026-01-24T12:35:00
categories:
  - rust
tags:
  - rust
---
## Rust实现最简单的Flappy Bird

《Rust游戏开发实战》中第3章简单小游戏

### 理解游戏循环

游戏循环首先会执行一次初始化操作，包括初始化显示窗口、图形设备以及其他的资源。此后，每当屏幕刷新一次显示时，它就会运行一次——通常以每秒30次、60次或者更高的频率运行。每一次循环，都会调用游戏程序中的tick()函数。  

游戏循环中做以下事情：

(1)配置应用程序、窗口以及图形设备。  
(2)轮询操作系统，以获取输入状态。     
(3)调用tick函数。tick()函数提供了游戏的实现逻辑。  tick函数每秒被调用次数为30或60，即30帧或60帧
(4)更新屏幕显示。一旦游戏程序的内部状态发生了更新，游戏引擎就需要更新屏幕显示  
(5)退出

### bracket-lib 工具库

bracket-lib实际上是一个用Rust语言编写的游戏开发软件库。它被设计为一个“简化版的教学工具”​，通过抽象屏蔽掉了游戏开发过程中各种复杂的事情，但保留了开发更复杂游戏所需要的概念。  

bracket-terminal是bracket-lib的显示组件。它提供了一个模拟的显示终端，并且可以在多种渲染平台上运行——从字符控制台到Web Assembly，包括OpenGL、Vulkan以及Metal 

**游戏循环运行的主要原理就是在每一帧中调用开发者编写的tick()函数。tick()函数本身对游戏一无所知，所以需要一种方式来存储游戏的当前状态（游戏状态，game state）​。任何需要在帧与帧之间保留的数据都存储在游戏的状态中。游戏状态代表了当前游戏进程的一个快照。**  

bracket-lib给用来存储游戏状态的类型定义了一个名为GameState的trait，GameState要求实现tick()函数，通过将引擎和所定义的State类型变量关联起来，这样bracket-lib才能知道tick()函数位于那种状态下。

main()函数需要初始化bracket-lib，描述期望创建的窗口类型以及游戏循环。

```rust
fn main() -> BError {
    let context = BTermBuilder::simple80x50()
        .with_title("Flappy Rust")
        .build()?;
    main_loop(context, State::new()) // 启动游戏主循环
}
```

context提供了一个窗口，用于和当前正在运行的bracket-terminal交互——可以通过它来获取鼠标的位置以及键盘输入，也可以给窗口发送绘图命令。

创建好终端窗口的实例后，你需要告诉bracket-lib执行`main_loop`函数启动游戏循环，并且在每一帧中调用tick()函数。可以把tick()函数看作连接游戏引擎和游戏程序本身的“桥梁”​。 

Bracket-lib会把字符转换为sprite图形来进行渲染显示，因此只能使用有限的字符集。显示在屏幕上的一个个字符其实是一张张图片——Bracket-lib库会根据发送给它的字符找到对应的图片，这些字符由Codepage 437字符集定义。  
#### 错误处理

如果代码中的很多函数都有潜在返回错误的可能性，那么充斥在代码中的unwrap()也会使得代码变得难以阅读。为每个可能失败的函数都使用match语句的做法同样会导致代码冗长且难以阅读。使用?操作符可以大幅度简化代码并使其易于阅读，唯一要求是你编写的这个函数必须也返回Result类型  

bracket-lib提供了一个名为BError的Result类型。把main函数的返回值改成BError类型就可以享受?操作符带来的便利  
#### 建造者模式

建造者模式发挥了函数链式调用的优点，可以把很多个参数选项分散到独立的函数调用中，相比在单一函数中写很长的参数列表，建造者模式可以提供更具可读性的代码。  

建造者模式发挥了函数链式调用的优点，可以把很多个参数选项分散到独立的函数调用中，相比在单一函数中写很长的参数列表，建造者模式可以提供更具可读性的代码 
### 游戏状态机

游戏通常运行在一种模态(mode)中。模态指定了在当前tick中游戏程序应该做什么事情，例如，显示主菜单或者游戏结束界面。在计算机科学中，这个概念有一个正式的名字叫作状态机(state machine)。在开发游戏之前先把游戏的基础模态框架定义出来是一个很好的做法，因为它可以作为后续要开发的游戏程序的“轮廓”​。  

```rust
enum GameMode {// 游戏状态枚举
    Menu,
    Playing,
    End,
}
```

游戏的tick()函数应该根据当前的模态指导程序的流程，而match语句非常适合做这件事。 

```rust
struct State {
    mode: GameMode,
    player: Player,
    frame_time: f32,
    obstacle: Obstacle,
    score: i32,
}
```

###  游戏角色

在Flappy Dragon游戏中，玩家要对抗重力作用、避开障碍物，才能生存下来。为了保持飞行状态，玩家需要按空格键来让飞龙扇动翅膀并获得向上的动力。为了实现这个逻辑，你需要存储飞龙当前的一些游戏属性。  

```rust
struct Player {// 玩家结构体
    x: i32,
    y: i32,
    velocity: f32, // 垂直方向的速度
}
```
    
玩家永远显示在屏幕的左侧。x坐标的数值实际上也代表了当前关卡的游戏进度。 虽然玩家角色在屏幕上的水平坐标不变，但你仍需知道当前关卡中（在世界坐标系下）玩家已经前进了多远。  

使用浮点数则允许使用小数形式的速度值——这可以带来流畅度大幅提升的游戏体验。  

你已经定义好了玩家角色对应的类型，现在需要把它的一个实例加入游戏状态变量中，并且在构造函数中将其初始化。此外，你需要增加一个名为frame_time的变量（它的类型是f32）​，这个变量用于累积若干帧之间经过的时间，通过它可以控制游戏的速度。  

ctx中有一个名为frame_time_ms的变量，它表示上一次tick()函数调用与本次tick()函数调用所隔的时间。将该变量累加到游戏状态的frame_time变量中，如果累加值超过了FRAME_DURATION常量，就运行物理引擎并且将frame_time变量清零。  

### 障碍物

为了得到障碍物在屏幕上的x坐标，你需要进行从世界坐标系到屏幕坐标系的转换。玩家角色在屏幕坐标系下的x坐标永远是0，但是在player.x中存放的是它在世界坐标系中的x坐标。由于障碍物的x坐标也是定义在世界坐标系下的，因此可以通过把障碍物的x坐标和玩家的x坐标相减的方式来获得障碍物在屏幕坐标系下的x坐标。  

```rust
struct Obstacle { // 障碍物结构体
    x: i32,
    gap_y: i32,
    size: i32
}
```

### 游戏效果


![](uploads/rust/flapygame.png)

### 代码实现

bracket-lib将开发者需要使用的一切功能都通过自身的prelude模块进行了导出，使用prelude模块可以让开发者在使用这个库时，不必每次都输入bracket-lib::prelude::。

```rust
use bracket_lib::prelude::*;

const SCREEN_WIDTH : i32=80;
const SCREEN_HEIGHT : i32=50;
const FRAME_DURATION : f32=75.0;

enum GameMode {// 游戏状态枚举
    Menu,
    Playing,
    End,
}

struct Player {// 玩家结构体
    x: i32,
    y: i32,
    velocity: f32, // 垂直方向的速度
}

impl Player {
    fn new(x: i32, y: i32) -> Self {
        Player {
            x,
            y,
            velocity: 0.0,
        }
    }

    fn render(&self, ctx: &mut BTerm) { // 每一帧渲染玩家，固定在屏幕的最左侧
        ctx.set(0, self.y, YELLOW, BLACK, to_cp437('@'));
    }

    fn gravitandmove(&mut self) {
        if self.velocity < 2.0 {
            self.velocity += 0.2; // 模拟空气阻力
        }
        self.y += self.velocity as i32;
        self.x += 1; // 水平移动，这里x是世界坐标系玩家的位置

        if self.y < 0 {
            self.y = 0;
            self.velocity = 0.0;
        } else if self.y > 49 {
            self.y = 49;
            self.velocity = 0.0;
        }
    }

    fn flap(&mut self) {
        self.velocity = -2.0; // 向上跳跃
    }
}

struct Obstacle { // 障碍物结构体
    x: i32,
    gap_y: i32,
    size: i32
}

impl Obstacle {
    fn new(x:i32, score:i32) -> Self {
        let mut rng = RandomNumberGenerator::new();
        let gap_y = rng.range(10, 40); // 障碍物间隙的垂直位置
        let size = i32::max(5, 20 - score); // 随着分数增加，障碍物间隙变小，最小为5
        Obstacle {
            x,
            gap_y,
            size
        }
    }

    fn render(&self, ctx:&mut BTerm, player_x:i32) {
        // 新障碍物的根据玩家世界坐标位置生成，为了把障碍物绘制在窗口，需要换算障碍物在窗口位置，
        // 这里的player_x是玩家的世界坐标，它会一直增加离障碍物越来越近, 而障碍物创建时self.x也是世界坐标
        // 因为玩家在屏幕上是固定位置0，所以障碍物在屏幕上的位置是self.x - player_x + 0
        let screen_x = self.x - player_x + 0; 
        if screen_x < 0 || screen_x >= SCREEN_WIDTH {
            return; // 不在屏幕范围内，不渲染
        }
        let half_size = self.size / 2; // 障碍物间隙的一半
        for y in 0..self.gap_y - half_size {
            ctx.set(screen_x, y, GREEN, BLACK, to_cp437('|'));
        }
        for y in self.gap_y + half_size..SCREEN_HEIGHT {
            ctx.set(screen_x, y, GREEN, BLACK, to_cp437('|'));
        }
    }

    fn hit_obstacle(&self, player: &Player) -> bool {
        let half_size = self.size / 2;
        let does_x_overlap = player.x == self.x;
        let player_above_gap = player.y < self.gap_y - half_size;
        let player_below_gap = player.y > self.gap_y + half_size;
        does_x_overlap && (player_above_gap || player_below_gap)  // 检测玩家是否在障碍物的间隙之外
    }
}

struct State {
    mode: GameMode,
    player: Player,
    frame_time: f32,
    obstacle: Obstacle,
    score: i32,
}

impl State {
    fn new() -> Self {
        State {
            player: Player::new(5, 25),
            frame_time: 0.0,
            mode: GameMode::Menu,
            obstacle: Obstacle::new(SCREEN_WIDTH, 0),
            score: 0,
        }
    }

    fn main_menu(&mut self, ctx: &mut BTerm) {
        ctx.cls();
        ctx.print_centered(5, "Welcome to Flappy Rust!");
        ctx.print_centered(8, "(Press P to Start)");
        ctx.print_centered(9, "(Press Q to Quit)");
        if let Some(key) = ctx.key {
            match key {
                VirtualKeyCode::P => self.restart(),
                VirtualKeyCode::Q => ctx.quit(),
                _ => {}
            }
            self.mode = GameMode::Playing;
        }
    }

    fn play(&mut self, ctx: &mut BTerm) {
        ctx.cls_bg(NAVY);
        self.frame_time += ctx.frame_time_ms;
        if self.frame_time > FRAME_DURATION { // 每75毫秒更新一次玩家下落加速状态
            self.frame_time = 0.0;
            self.player.gravitandmove();
        }
        
        if let Some(VirtualKeyCode::Space) = ctx.key {// 按空格键让玩家跳跃
            self.player.flap();
        }
        self.player.render(ctx); // 渲染玩家
        ctx.print(0, 0, "Press Space to Flap.");
        ctx.print(0, 1, &format!("Score: {}", self.score));
        self.obstacle.render(ctx, self.player.x); // 渲染障碍物
        if self.player.x > self.obstacle.x { // 玩家通过障碍物，生成新的障碍物
            self.score += 1;
            self.obstacle = Obstacle::new(self.player.x + SCREEN_WIDTH, self.score); // 新的障碍物生成在相对玩家位置的屏幕右侧外
        }
        if self.player.y >= SCREEN_HEIGHT - 1 || self.obstacle.hit_obstacle(&self.player){ // 玩家触底，游戏结束
            self.mode = GameMode::End;
        }

    }

    fn restart(&mut self) {
        self.player = Player::new(5, 25);
        self.frame_time = 0.0;
        self.score = 0;
        self.obstacle = Obstacle::new(SCREEN_WIDTH, 0);
        self.mode = GameMode::Playing;
    }

    fn game_over(&mut self, ctx: &mut BTerm) {
        ctx.cls();
        ctx.print_centered(5, "Game Over!");
        ctx.print_centered(6, &format!("You earned {} points", self.score));
        ctx.print_centered(8, "(Press P to Restart)");
        ctx.print_centered(9, "(Press Q to Quit)");
        if let Some(key) = ctx.key {
            match key {
                VirtualKeyCode::P => self.restart(),
                VirtualKeyCode::Q => ctx.quit(),
                _ => {}
            }
        }
    }
}

impl GameState for State {
    fn tick(&mut self, ctx: &mut BTerm) {
        match self.mode {
            GameMode::Menu => self.main_menu(ctx),
            GameMode::Playing => self.play(ctx),
            GameMode::End => self.game_over(ctx),
        }
    }
}

fn main() -> BError {
    let context = BTermBuilder::simple80x50()
        .with_title("Flappy Rust")
        .build()?;
    main_loop(context, State::new()) // 启动游戏主循环
}
```