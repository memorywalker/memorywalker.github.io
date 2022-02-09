---
title: Github study
date: 2020-02-07 20:25:49
tags: github; git
---


## Github

当多人合作时，可以每个人各自创建一个分支，每个分支都有明确的名称，做完自己的开发后，合并到一起

### 远程仓库

远端仓库是存在远端服务器或PC上的git仓库，可以使用URL或文件系统的路径来访问一个远程仓库

可以把本地的repo的分支同步到remote repo，一个本地的repo可以关联多个远端repo

### remote

`git remote`可以查看当前关联的remote repo的路径，一般使用origin作为主干的remote repo的名称

关联一个remote repo，在本地的repo目录下，执行

`git remote add origin https://github.com/memorywalker/workflow.git`

其中的origin只是一个惯例，也可以使用任意一个名称来代表远端repo，然后使用

`git remote -v`查看当前关联的remote repo是否正确

`git remote rename newname oldname`更改一个remote repo的别名

### push

`git push origin master`把本地的master分支发送到名为origin的远端repo，会在远端创建一个master分支

```shell
To https://github.com/memorywalker/workflow.git
 * [new branch]      master -> master
```

执行`git log --oneline --all`可以看到当前本地更新的远端分支在哪个commit上，其中的`origin/master`称作追踪分支，表示一个远端分支当前指向当前的哪个commit

```shell
0f40286 (HEAD -> master, origin/master, backup) change call of duty
```

### pull

`git pull origin hexo`从名为origin的远端更新hexo分支的commit到本地，pull会合并远端分支的更改到本地

### fetch

当本地的更改和远端的commit有冲突时，可能不需要git自动合并remote的更改到本地，此时需要先把远端的更改下载到本地，在本地手动合并冲突后，再把本地的push到远端

` git fetch origin master`从名为origin的远端下载master分支到本地，但是不合并到本地的master分支

```shell
$ git log --oneline --all
f85bd96 (origin/master) add h2 style
0f40286 (HEAD -> master, backup) change call of duty
```

如果要把已经下载下来的合并到本地分支，需要本地执行merge命令

`git merge origin/master`,在本地把冲突处理

### shortlog

`git shortlog`可以查看每一个提交者提交了多少次以及每次提交信息，默认使用作者的名称字母顺序，可以增加`-n`安提交次数降序排列，`-s`只显示提交次数，不显示提交信息

### log

`git log --author=xxx `只显示作者名字以xxx开始提交的日志，如果名字中有空格，需要使用""包住

`git log --grep=bug`和`git log --grep bug`过滤commit的信息中有bug的commit，这里grep的规则和shell的grep相同，如果有空格也需要""包住

### rebase

rebase可以把多个commit合并到一起，如果和多人一起工作，不要把已经push过的commit执行rebase，这样会导致其他人本地的和库里面的不一致，合并起来很麻烦。

`git rebase -i HEAD~3`从`HEAD~3`的位置重新创建一个base，这个commit之后的会合并到一起，之后`git log`不会看见已经合并的这些commit，`-i`标识交互的方式进行rebase

在执行rebase之前可以先创建一个backup分支，避免rebase之后被合并的commit被删除了无法恢复

```shell
*   c4f25cd (HEAD -> backup, master) change h2 style
|\
| * f85bd96 (origin/master) add h2 style
* | ff309fe add h2 style local
|/
* 0f40286 change call of duty
* 65d78c2 Revert "change title"
* ee4190c change title
```

执行`git rebase -i HEAD~3`后

```shell
pick 0f40286 change call of duty
pick ff309fe add h2 style local
pick f85bd96 add h2 style

# Rebase 65d78c2..c4f25cd onto 65d78c2 (3 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
```

修改其中的内容，从下向上依次是最早的commit，前缀改为s，说明要把这个commit合并到它的上一个commit，而r对这次提交重新写commit信息，作为最后rebase的新的commit的信息

```shell
r 0f40286 change call of duty
s ff309fe add h2 style local
s f85bd96 add h2 style
```

保存文件后，会提示编辑commit信息

合并后65d78c2现在是master的base，中间的其他commit都没有了，不过backup分支还有备份

```
* fc0772e (HEAD -> master) add h2 style
| * 9848bbf (readme) add readme file
| *   c4f25cd (backup) change h2 style
| |\
| | * f85bd96 (origin/master) add h2 style
| * | ff309fe add h2 style local
| |/
| * 0f40286 change call of duty
|/
* 65d78c2 Revert "change title"
* ee4190c change title
```

### Github

#### fork

拷贝一份其他人的repo到自己的账户

#### push

> remote: Support for password authentication was removed on August 13, 2021. Please use a personal access token instead.
> remote: Please see https://github.blog/2020-12-15-token-authentication-requirements-for-git-operations/ for more information.

现在github不再使用用户名密码作为验证，而使用token，这个token在` [Personal Access Tokens (github.com)](https://github.com/settings/tokens) `生成，在生成的页面会显示一次，需要自己保存好，每一个token可以有不同的权限和有效期设置

本地push时，输入用户名后，提示输入密码要用这个新生成的token（一串字符）

#### issue

如果要给公共库提交更改，要先查看库的贡献说明文档；查看issue列表是否有类似的问题，咨询库的所有者是否有人在处理这个问题、自己是否可以处理，避免浪费工作时间；是不要提交一个issue来追溯这个更改

github的issue不只是bug，可以是项目相关的任何问题，可以把一个issue指派给一个人或一个版本，一个issue下面可以评论，你也可以订阅这个issue，只要有变化，你都会收到通知

如果一个项目有` CONTRIBUTING.md `这个文件，在给项目新建issue时，会在页面的最下提示Remember, contributions to this repository should follow its [contributing guidelines](https://github.com/memorywalker/workflow/blob/master/CONTRIBUTING.md). 链接到项目的贡献说明文档

master分支作为默认的分支一般用来放所有的commit，而更改一个故障可以创建一个topic分支，分支的命就可以是bug-xxx之类，不要在master分支做自己的更改

尽量经常提交小的commit，一个commit的更改一定不能太多，比如十几个文件，几百行代码，因为管理者在合并你的代码时，可能会觉得其中的一部分时合适的，而另一部分不合适，如果全部放在一个commit里，无法单独更改

做了更改之后，不要忘记更多readme文件

#### pull request

当你在forked的项目上修改了一个故障，此时需要原始的项目维护者从你forked的项目pull这个更改到原始的项目上时，做的一个request

常规流程：

1. fork一个原始项目AA到自己的账户下
2. 把forked的项目下载到本地，并创建一个topic分支进行更改
3. 把topic分支的更改push到自己的账户
4. 在GitHub创建一个pull request并选择更改的topic分支

#### watch && star

watch:当项目有任何的变化都会通知到你的邮箱，如果你是项目的维护者，需要这个

star:在自己的主页可以看到项目的更改，但是不会主动通知

#### 与源项目同步

fork的项目在本地更改后，原始的项目可能已经更新了内容，但是还是需要把源项目的更改同步过来的

1. 在本地的项目中增加源项目作物一个remote repo

   `git remote add upstream https://github.com/udacity/course-collaboration-travel-plans.git`

   `upstream`通常作为原始项目的remote的别名

2. `git remote -v`查看本地的项目应该是关联了两个remote的repo

3. `git fetch upstream master`从源项目获取最新的更改

4. `git checkout master`本地的分支切换到master分支

5. `git merge upstream/master`合并远端upstream的master分支到本地的master分支

6. `git push origin master`把最新的master推到自己的GitHub的项目的master上

### Reference

[http://www.firsttimersonly.com/](http://www.firsttimersonly.com/ )

[up for grabs](https://up-for-grabs.net/#/)


### Vocabulary

 defacto  事实上; 事实; 事实上的; 实际上; 实际上的 

 substantial  大量的; 价值巨大的; 重大的; 大而坚固的; 结实的; 牢固的 

 `a11y` stands for "accessibility". In the word "accessibility", there are eleven letters between the `a` and the `y`, so it gets shortened to just `a11y` 

 squash  压软(或挤软、压坏、压扁等); 把…压(或挤)变形; (使) 挤进; 塞入; 打断; 制止; 去除; 粉碎;  墙网球; 壁球; 果汁饮料; 南瓜小果 



