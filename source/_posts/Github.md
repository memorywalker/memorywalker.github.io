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



### Github

#### fork

拷贝一份其他人的repo到自己的账户

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


### Vocabulary

 defacto  事实上; 事实; 事实上的; 实际上; 实际上的 

 substantial  大量的; 价值巨大的; 重大的; 大而坚固的; 结实的; 牢固的 

 `a11y` stands for "accessibility". In the word "accessibility", there are eleven letters between the `a` and the `y`, so it gets shortened to just `a11y` 



