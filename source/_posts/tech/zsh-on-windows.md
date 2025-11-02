---
title: Windows安装zsh终端
date: 2025-11-02T23:24:00
categories:
  - tech
tags:
  - git
  - windows
---
## Windows安装zsh

### 安装Git Bash

https://git-scm.com/install/windows 下载并安装Git，安装后就会附带Git Bash

### 安装zsh

https://packages.msys2.org/packages/zsh?repo=msys&variant=x86_64 下载[zsh-5.9-4-x86_64.pkg.tar.zst](https://mirror.msys2.org/msys/x86_64/zsh-5.9-4-x86_64.pkg.tar.zst) 

zst文件可以使用 [7-Zip-zstd](https://github.com/mcmilk/7-Zip-zstd) 解压后，其中其中tar包中的所有文件覆盖到`git-bash.exe`所在的目录中，过程中会提示覆盖文件，覆盖即可。
**备注**：我解压了最新的5.9-4版本，覆盖之后，git bash就打不开了，所以到网上找了别人解压后的[5.9-2的版本](https://www.seepine.com/git/oh-my-zsh/zsh-5.9-2-x86_64.pkg.zip)是可以正常运行。

#### 配置git bash默认使用zsh

编辑用户目录下的`.bashrc`文件，添加以下内容

```bash
if [ -t 1 ]; then
  exec zsh
fi
```

### 安装oh-my-zsh

安装命令  
`sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"`

github如果不能正常访问，可以使用代理安装，在原来地址前加上代理的网址前缀 
`sh -c "$(curl -fsSL https://gh.felicity.ac.cn/https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"`

### 配置oh-my-zsh

zsh的配置文件`.zshrc`也在用户目录中
#### 配置主题

配置文件中`ZSH_THEME="robbyrussell"`默认使用的主题为**robbyrussell**
配置项的上面有详细说明，可以配置为随机主题`ZSH_THEME="random"`

主题可以在[Themes](https://github.com/ohmyzsh/ohmyzsh/wiki/Themes)这里查看

修改默认主题`robbyrussell`显示完整路径，在`C:\Users\Edison\.oh-my-zsh\themes`主题目录中找到`robbyrussell.zsh-theme`文件，修改

```bash
%{$fg[cyan]%}%c%{$reset_color%} 为 %{$fg[cyan]%}%d%{$reset_color%}，

如果只需要显示最后两级目录，只需在d前增加数字2，改为这样%{$fg[cyan]%}%2d%{$reset_color%}
```

#### 配置插件

1. **自动补全**
自动补全插件可以灰显方式提示最近使用的过命令，按方向键右键就可以快速补全
执行以下命令
 `git clone https://gh.felicity.ac.cn/https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions`
或者 到`C:\Users\Edison\.oh-my-zsh\custom\plugins`目录下，把`https://github.com/zsh-users/zsh-autosuggestions`克隆到这个目录中。

在配置文件`.zshrc`文件中找到`plugins=(git)`，默认只有git插件激活了，增加`zsh-autosuggestions`
`plugins=(git zsh-autosuggestions)`

2. **语法高亮**
语法高亮插件在输入的shell命令如果错误时，会用红色标识，正常的命令会用绿色标识。

执行以下命令下载插件到`C:\Users\Edison\.oh-my-zsh\custom\plugins`自定义插件目录下
`git clone https://gh.felicity.ac.cn/https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting`

配置激活插件
`echo "source ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh" >> ${ZDOTDIR:-$HOME}/.zshrc`
这条语句会在配置文件`.zshrc`文件中的最后一行添加`source ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh`，使得zsh每次启动时都会执行`zsh-syntax-highlighting.zsh`这个脚本。

也可以通过插件的方式 `plugins=(git zsh-autosuggestions z zsh-syntax-highlighting)` 激活高亮插件

3. **快速目录切换**
在配置文件中增加z插件激活`plugins=(git zsh-autosuggestions z)`。后续如果想快速切换目录，可以执行`➜  ~ z pl`就可以快速切换到刚刚访问过的目录名称中有pl字母的目录，例如刚刚切换到plugins目录，就会直接切换过去。








