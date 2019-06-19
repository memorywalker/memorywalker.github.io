---
title: GitPages+Hexo+CI 自动部署个人主页
date: 2019-06-19 22:02:50
tags: blog
---

## 

现在已经习惯了使用Markdown写日志了，个人blog还是要坚持记录，WordPress平台的服务器资源总是不稳定，所以还是恢复很久之前使用gh-pages搭的主页。原来这里只是放了一篇模板文件 ORz

### HEXO

之前使用了HEXO作为静态blog的框架，虽然Github官方支持的是Jekyll，但是之前创建仓库时用的Hexo，还想继续用原来的仓库，就不再调整了

#### 安装

1. 安装nvm 

`$ sudo apt install curl`

`$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash`

2. 安装node.js `$ nvm install stable`

3. 使用npm安装Hexo `$ npm install -g hexo-cli`

4. 非空目录下初始化工程 `$ hexo init .`

5. 安装相关插件 `$ npm install` 

最终得到如下结构目录
```
.
├── _config.yml   配置文件
├── package.json  程序信息
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts  源码目录，md文件放在这里
└── themes
```

#### 写文章

* 执行命令新建一个文章

`$ hexo new "post title with whitespace"`

在`source/_post/`下会自动生成md文件

打开后有文件基本信息，就可以正常写内容了

* 生成文章 

`$ hexo generate`


### Github部署

### CI 自动发布

如果本地没有node.js的环境，此时如果需要发布文章，还要搭建完整的开发环境，使用TravisCI可以自动编译github上的工程，并把结果进行发布
https://www.travis-ci.org/ 使用github账号可以直接登陆

1. 在自己的所有工程列表中，打开需要自动部署的工程，并点击Settings
2. Settings--General: 只需要打开`Build pushed branches`,其他两个保持关闭
3. Environment Variables中增加一个Name 为GH_TOKEN，值为自己的Github Personal access Token
4. Github的个人设置中，进入`Developer settings`，在`Personal access tokens`中新建一个token，勾选Repo和user两个项，把自动产生的一段token放到刚刚的环境变量value中
5. 在博客的根目录新建`.travis.yml`文件，内容为

```yml
language: node_js
node_js: stable

# assign build branches
branches:
  only:
    - hexo # this branch will be build

# cache this directory
cache:
  directories:
    - node_modules

# S: Build Lifecycle
before_install:
  - npm install -g hexo-cli # install hexo
  #- git clone https://github.com/renyuzh/hexo-theme-next.git themes/next 

install:
  - npm install # install by package.json

script:
  - hexo generate

after_success:
  - git config --global user.name "memorywalker"
  - git config --global user.email "eddy.wd5@gmail.com"
  - sed -i "s/gh_token/${GH_TOKEN}/g" _config.yml #使用travisCI中配置的token替换掉_config.yml中对应的占位符
  - hexo deploy
# E: Build LifeCycle
```

6. 修改hexo的配置文件，把原来的自动部署的repo地址更新为https的
```yml
deploy:
  type: git 
  repo: https://gh_token@github.com/memorywalker/memorywalker.github.io.git
  branch: master
```

7. 把更新的文件push到博客源码分支hexo

8. 在`https://www.travis-ci.org/memorywalker/memorywalker.github.io`可以查看编译运行情况

[基于TravisCI自动化部署Hexo博客到Github](https://www.jianshu.com/p/7e47840eee26)

