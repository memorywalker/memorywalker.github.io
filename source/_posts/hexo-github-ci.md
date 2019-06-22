---
title: GitPages+Hexo+CI 自动部署个人主页
date: 2019-06-19 22:02:50
tags: blog
---


现在已经习惯了使用Markdown写日志了，个人blog还是要坚持记录，WordPress平台的服务器资源总是不稳定，所以还是恢复很久之前使用gh-pages搭的主页。原来这里只是放了一篇模板文件 ORz

### HEXO

之前使用了HEXO作为静态blog的框架，虽然Github官方支持的是Jekyll，但是之前创建仓库时用的Hexo，还想继续用原来的仓库，就不再调整了

#### 安装

1. 安装nvm 

`$ sudo apt install curl`

`$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash`

提高npm的安装速度可以使用taobao的镜像服务，地址为[cnpm](http://npm.taobao.org/)，先安装
`$ npm install -g cnpm --registry=https://registry.npm.taobao.org`
后续使用`cnpm install xxx --save`来安装插件

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

* 本地预览

`$ hexo server`
系统提示服务器的地址`http://0.0.0.0:4000/memorywalker/`
```
INFO  Start processing
INFO  Hexo is running at http://0.0.0.0:4000/memorywalker/. Press Ctrl+C to stop
```

* 执行命令的过程中增加`--debug`选项可以输出更多的调试信息，方便定位原因例如 `hexo s --debug` 

#### 升级Hexo
1. 升级全局的hexo`npm i hexo-cli -g`
2. 新建一个目录，`$ hexo init .`创建一个新的开发环境
2. 删除原来目录中的`node_modules`和`themes`目录，把并把新目录的这两个目录复制到原来的目录中
3. 使用比较工具合并`_config.yml`文件的内容
4. 使用比较工具`package.json`文件的内容，把新的文件覆盖的旧目录后，把以前需要的插件再补充安装，例如git部署插件就需要重新安装`npm install hexo-deployer-git --save`

#### 安装Next主题

1. 把next主题下载一份到工程的themes目录下
`$ git clone https://github.com/theme-next/hexo-theme-next themes/next`

2. 修改工程的`_config.yml`中的`theme: landscape` 为 `theme: next`

3. 执行`hexo clean`清除原来的缓存，`hexo s`生成新的文件并进行预览

4. 升级主题 `$ cd themes/next` and then `$ git pull`

5. 安装next主题后，使用`Travis-CI`自动部署会出现访问页面时主题用到的资源无法加载，需要修改原来项目`_config.yml`中的url如下:
```yml
url: http://memorywalker.github.io
root: /
```

* 安装本地搜索插件 

`cnpm install hexo-generator-searchdb --save`

修改`themes\next\_config.yml`找到`local_search`，设置为true

修改项目的`_config.yml` 添加如下：
```yml
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
  content: true
```




### Github部署

GitHub Pages是针对个人提供的页面，一个用户只能有一个这样的仓库。这个仓库的名称必须是`用户名.github.io`，对应的访问网址也是`用户名.github.io`

新建`用户名.github.io`的仓库后，在这个仓库的Setting页面有GitHub Pages配置

>GitHub Pages is designed to host your personal, organization, or project pages from a GitHub repository.

这个配置项中说明了发布地址，以及用户page必须放在master分支，master分支最终只会有hexo编译转换出来的静态博客的网页文件，它的文件都来自`hexo g`产生的`public`

在本地的hexo目录下新建一个Hexo分支，这个分支用来保存博客的源码程序，这个分支中只把上面的Hexo的框架文件和目录添加到分支，对于`node_modules`node的插件文件,`public`生成的发布文件,`db.json`这些文件不需要添加到分支更新到仓库。

* 安装git部署插件 `$ npm install hexo-deployer-git --save`
* 修改hexo的配置文件`_config.yml`,其中增加

```yml
deploy:
  type: git   
  repo: git@github.com:memorywalker/memorywalker.github.io.git
  branch: master
  message: [message]  #leave this blank
```

* 执行`$ hexo deploy`,hexo会自动把public的文件push到github的master分支

以后每次写完markdown文件后，只需要`$ hexo generate --deploy`，在生成后自动发布

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
    - themes

# S: Build Lifecycle
before_install:
  - npm install -g hexo-cli # install hexo
  - git clone https://github.com/theme-next/hexo-theme-next themes/next

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

