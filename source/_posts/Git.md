---
title: Git study
date: 2020-02-05 20:30:49
tags: git
---

## Git 

/git/ 

[BOOK](https://git-scm.com/book/en/v2)

### Terminology  

/tɜːrmɪˈnɑːlədʒi /  (某学科的) 术语; 有特别含义的用语; 专门用语 

 **version control system** (abbreviated as **VCS**) 

 **source code manager** (abbreviated as **SCM**) 

 **commit**  保存一份当前项目的state到git中，可以看做游戏保存当前进度

 **Repository / repo** 一个仓库中包含了项目的所有文件，由commit组成

 **Working Directory** 本地的工作目录

 **checkout** 把repo中的所有文件拷贝一份到本地目录

 **staging area** as a prep table where Git will take the next commit. Files on the **Staging Index** are poised to be added to the repository 

 **branch** 分支 游戏中保存一个新的存档，然后就可以选择不同的结局，在Half Life结尾G Man给你选择前可以新建一个存档位置，可以选择不为他打工



**Working Directory**  -(add)-> **staging area** -(commit)-> **Repository**



### Config

1. 右键打开Git bash，直接输入`cd`，进入**home**目录

2. `start .` 在资源管理器中打开目录

3. 再打开的文件中，右键点收藏夹，将当前文件添加到收藏夹，方便以后打开这个目录

4. 把下载的配置文件中的`bash_profile`和文件夹`udacity-terminal-config`拷贝到根目录

5. 由于windows不支持修改文件名为.开始的名字，需要在命令提示符下使用`mv`命令实现

   `$ mv bash_profile .bash_profile`

   `$ mv udacity-terminal-config .udacity-terminal-config`

6.  重新打开一个bash窗口，点击左上角，option，设置前景色为黑色，背景色为白色   

7. 执行以下命令进行全局配置

```shell
# sets up Git with your name
git config --global user.name "<Your-Full-Name>"

# sets up Git with your email
git config --global user.email "<your-email-address>"

# makes sure that Git output is colored
git config --global color.ui auto

# displays the original state in a conflict
git config --global merge.conflictstyle diff3

git config --list

# git work with sublime editor
git config --global core.editor "'C:/Program Files/Sublime Text 2/sublime_text.exe' -n -w"

# git work with VS Code
git config --global core.editor "code --wait"
```

### 基本使用

#### init一个Repo

1. 新建一个目录并进入到新建目录中`mkdir -p udacity-git-course/new-git-project && cd $_ `
2. 执行`git init`，会在当前目录下创建一个repo，`.git`中就是这个repo的目录

Repo中的内容

* **config file** - where all *project specific* configuration settings are stored. 
*  **description file** - this file is only used by the GitWeb program 
*  **hooks directory** - this is where we could place client-side or server-side scripts that we can use to hook into Git's different lifecycle events 
*  **info directory** - contains the global excludes file 
*  **objects directory** - this directory will store all of the commits we make 
*  **refs directory** - this directory holds pointers to commits (basically the "branches" and "tags") 

#### clone一个Repo

clone可以创建一个现有项目的完全相同的复制

执行`git clone https://github.com/udacity/course-git-blog-project `会创建一个新的项目目录`course-git-blog-project `在当前目录中

执行`git clone http://xxx/project newName`可以在克隆时直接换一个本地的目录名称

#### status

`git status`查看当前repo的状态，应该在执行每一个git的命令后都查看一下status



#### log

`git log`查看所有commit历史记录

输出的内容在**Less**中相同

- 下翻
  - `j` or `↓` 下翻一行
  - `d` 下翻半屏
  - `f` 下翻一屏
- 上翻
  - `k` or `↑` 上翻一行
  - `u` 上翻半屏
  - `b` 上翻一屏
- 退出 press `q` to **quit** 

` git log --oneline` 简化显示log信息

`git log --stat`显示每一个commit的汇总信息，stat是 statistics 的缩写

`git log -p` p是patch的缩写，显示每个文件具体改了哪些内容

` git log -p --stat -w `可以组合使用标记，`-w`不显示空白行的更改

git以行为单位对文件的更改进行追踪

```diff
diff --git a/index.html b/index.html  (正在显示的文件)
index 0381211..43f5b28 100644 （更改前的前后的这个文件的hash）
--- a/index.html  （指明旧的文件）
+++ b/index.html  （指明新的文件）
@@ -15,83 +15,85 @@ （-标识旧文件，从15行开始共83行，+标识新文件，15行开始，共85行）
         <h1>Expedition</h1>
     </header>

-    <main>   （旧文件删除的行）
-        <h2 class="visuallyhidden">Articles</h2>
+    <div class="container">  （新文件增加行）
+        <main>
+            <h2 class="visuallyhidden">Articles</h2>
```

* `git log -p fdf5493`显示fdf5493和这个commit之前的所有log

* `git show [SHA]`查看指定的一次提交的信息，默认附带了`-p`标记，如果要加`--stat`会把默认的`-p`标记去掉，要手动加上`-p`, `-w`不显示对空白行的更改 `git show --stat -p 8d3ea36`

#### add

将文件从**work directory**加入**staging index**

* `git add index.html`增加一个文件到staging index，多个文件用空格分隔开
* `git rm --cached index.html` 删除一个staged的文件
* `git add .`把当前目录下的所有文件增加到staging index

#### commit

`git commit`会打开配置的默认编辑器，当保存文件，关闭编辑器后，数据才会提交

`git commit -m "Initial commit"`提交信息使用`-m`

每次提交应该只有一个重点，记录一个单位的更改，只是更改项目的一个方面

一次提交不能包含不相关的更改

##### 提交信息

* 信息简短，不超过60个英文单词
* 解释提交内容做了什么，而不是为什么或怎么做的
* 不要解释为什么做了这个更改
* 不要解释怎么做了更改
* 不要使用and，说明你提交了多个更改
* 写完简短的信息后，可以换行增加一个空行，再写详细的更改原因，方便`git log --oneline`

udacity的[commit style guide](https://udacity.github.io/git-styleguide/ )

#### diff

用来查看当前没有commit的更改

#### gitignore

在和`.git`目录同级的目录下使用`touch .gitignore`新建`.gitignore`文件用来屏蔽那些不需要版本管理的文件

##### globbing规则

* 空行用来分隔
* `#`标识注释
* `*`匹配0或多个字符
* `?`匹配1个字符
* `[abc]`匹配a, b, or c
* `**`匹配嵌入的目录 `a/**/z`匹配`a/z`,`a/b/z`, `a/b/c/z`

#### tag

tag用来标识一个特殊的版本，比如beta1.0，它和一个commit关联起来

`git tag -a v1.0`会以当前的commit创建一个tag并打开编辑器等待输入tag的备注信息，`-a`指明创建一个annotated tag，建议始终带有a选项的tag，包含更多的信息，如果不带a，只是一个轻量级的tag，没有创建人和创建日期信息

`git tag`列出当前repo的所有tag，使用`git log`可以看到当前的tag信息

`git tag -d v1.0`删除tag v1.0

`git tag -a v1.0 9a2e3bf`指定commit创建一个tag

#### branch

一个Tag永久性的指向一个commit，一个branch会移动到最后的一个commit

master是git给的默认branch，head指向当前活动的branch

`git branch`列出当前的所有分支，星号标识的是当前分支

`git branch feature`以当前的commit创建一个名为feature的分支

`git branch feature SHA`以SHA对应的commit创建一个名为feature的分支

`git checkout master`切换到master分支，checkout可以在多个branch之间切换，让head指向当前的分支。这个命令会：

1. 删除当前工作目录下的所有被git管理的文件（所有已经commit到repo中的文件），没有被add或commit的文件会保持不变
2. 从repo中取出指定分支的文件到当前工作目录

`git branch -d feature`删除名为feature的分支，当前活动的分支不能被删除，如果一个分支上有commit是只有这个分支才有的，还没有合并到其他分支，也不能删除；如果要强制删除这个有自己的commit的分支，使用`git branch -D feature`

`git checkout -b footer master`基于master分支创建footer分支，并切换到footer分支

`git log --graph --all --oneline` graph用来显示log最左侧的分支路径线all参数用来显示repo中的所有分支

#### merge

把分支的更改进行合并，git可以自动合并不同分支的更改

* 普通merge ： 如果两个分支有差异的内容，把另一个分支的内容合并到当前的分支，此时merge也是一次commit，需要提供message，而且git已经提供了默认的message
* fast-forward merge 如果一个分支newfeature已经在master的前面（在master的基础上已经有了新的更改，但是master一直没有更改），此时要把它合入master分支，在合并的时候，只是把master指向newfeature的commit即可，并不需要一次新的commit

`git merge name-of-branch-to-merge-in`把另一个分支合入当前的分支，例如`git merge sidebar`

##### 冲突处理

git以文件中的一行为单位作为文件改变的标识，当两个分支中对同一个文件的同一行都有修改，在自动merge的时候，就不能自动选择用哪一个分支的了

```shell
$ git merge head-update
Auto-merging index.html
CONFLICT (content): Merge conflict in index.html
Automatic merge failed; fix conflicts and then commit the result.
```

此时执行`git status`会提示

```shell
On branch master
You have unmerged paths.
  (fix conflicts and run "git commit")
  (use "git merge --abort" to abort the merge)

Unmerged paths:
  (use "git add <file>..." to mark resolution)
        both modified:   index.html
```

此时文件已经被改动，并且有标记哪些部分是冲突的

```html
    <header>
<<<<<<< HEAD  本地分支当前内容
        <h1>Future</h1>
||||||| b27a903 合并前的上一次的原始内容
        <h1>Expedition Future</h1>
======= 合并内容的结束行标记
        <h1>Past</h1>
>>>>>>> head-update 合入的分支的结束标记
    </header>
```

在编辑器中直接修改文本内容为最终需要的内容，保存后提交，可以在提交之前执行`git diff`查看更改的内容，避免把标记没有删除也提交上去

#### amend

`git commit --amend`修改最近一次的commit，而不会产生新的commit。

如果当前已经没有需要commit的内容，则会弹出编辑commit message的编辑器，修改message的内容

如果有遗漏的文件忘记修改，可以修改文件后并执行add来stage文件，执行`git commit --amend`让上次的commit增加新的文件

#### revert

revert是对一次commit的恢复，因此也是一次新的commit

```shell
$ git revert ee4190c
[master 65d78c2] Revert "change title"
 1 file changed, 1 insertion(+), 1 deletion(-)
Moon (master) newrepo
$ git log --oneline
65d78c2 (HEAD -> master) Revert "change title" #新的一次提交
ee4190c change title

```



#### reset

 reset从repo中删除一个commit，git会在删除数据前保存所有的信息30天，可以使用`git reflog`

在执行reset之前可以对当前的commit创建一个backup的新分支用来备份commit的数据`git branch backup_somework`。需要恢复时，`git merge backup`即可

`git reset <reference-to-commit>`把Head指向reference commit，删除中间的commit，把已经commit的数据放入staging index，把staged的数据变为unstaged

`git reset --mixed HEAD^ `默认的选项，把当前commit的内容回退到work directory，变为unstaged状态

`git reset --soft HEAD^ `把当前commit的内容回退到staging index

`git reset --hard HEAD^ `把当前commit的内容放入stash

`git checkout -- <filename>`撤销当前工作目录中filename文件的所有更改

##### Relative Commit References

相对commit引用, `HEAD`指向当前commit，`^`指向当前的父commit，`~`指向第一层父commit

```shell
HEAD^ = HEAD~ = HEAD~1
HEAD^^ = HEAD~2
```

一个merge的commit有两个父commit，`^`指向执行`git merge`分支的父commit，`^2`指向合并过来的分支的父commit

```shell
* 9ec05ca (HEAD -> master) Revert "Set page heading to "Quests & Crusades""
* db7e87a Set page heading to "Quests & Crusades"
*   796ddb0 Merge branch 'heading-update'
|\  
| * 4c9749e (heading-update) Set page heading to "Crusade"
* | 0c5975a Set page heading to "Quest"
|/  
*   1a56a81 Merge branch 'sidebar'
```

 `HEAD^^^` 指向 `0c5975a` ，只有当前分支路径上带`*`的commit都是这个分支的

 `HEAD^^^2` 指向 `4c9749e` 

### Vocabulary

* sneak / sniːk /  偷偷地走; 溜; 偷偷地做; 偷带; 偷拿; 偷走(不重要的或小的东西);  突然的; 出其不意的 ; 打小报告的人，告状者(尤指儿童);

 Wanna *have a sneak peak of* the next lesson (偷偷看一下)

* intro  介绍; (尤指) 前奏，前言，导言 
* outro  结尾部分 
*  globbing   通配符; 文件名扩展; 文件名代换; 展开 
*  annotated  给…作注解(或评注) 
*   delve  /delv/  (在手提包、容器等中) 翻找;  delve into her mother's past探究母亲的过去
*  nitty  尼堤; 多虱卵的; 很紧甚至有些紧弱; 
*   gritty  含沙砾的; 沙砾般的; 有勇气的; 坚定的; 坚毅的; (对消极事物的描述) 逼真的，真实的，活生生的;  The sheets fell on the *gritty* floor  床单掉到满是沙砾的地板上 
*   nitty gritty  本质; 实质; 基本事实; The city's newspapers still attempt to get down to the *nitty* *gritty* of investigative *journalism*  该市报纸仍在试图厘清调查性新闻的实质 
*   asterisk / ˈæstərɪsk / 星号(置于词语旁以引起注意或另有注释) 
*   nerve-wracking  令人焦虑的; 使人十分紧张的 
*   grins  露齿而笑; 咧着嘴笑; 龇着牙笑 
*   giggles  咯咯笑; 傻笑; 趣事; 玩笑; 可笑的事; 止不住的咯咯笑 
*   divergent  有分歧的; 不同的; 相异的; 