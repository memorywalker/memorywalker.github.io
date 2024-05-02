---
title: Vue3 简单使用
date: 2024-05-02 22:01:49
categories:
- programming
tags:
- web
- vue
- frontend
---

## Vue 3简单使用

vue3的官方文档  [简介 | Vue.js (vuejs.org)](https://cn.vuejs.org/guide/introduction.html)  对于从来没有用过vue的初学者有很多看不懂的内容。

要不看视频教程B站上有很多，或者系统的找一个电子书学习一下基础知识。

我参考了`Fullstack Vue The Complete Guide to Vue.js and Friends`这本书，用例子的方式一步一步的引入vue的各种特性。

官方教程上来就用vue-cli来创建一个vue应用，模版程序里面有多个组件，但这时新手对于组件一点概念也没有，也不知道应用程序目录下的那一堆程序文件分别是什么作用。

### JavaScript

2012年做web开发的时候用的还是JQuery，里面的各种快捷的操作和取元素以及结合ajax处理客户端事件操作DOM对象已经很方便了，现在看了技术发展会提供越来越多的框架和语法糖，提高开发效率，硬件配置的提升，弱化了性能的要求。JavaScript语言规范ECMAScript(ES)也有了很大的发展。

2015年完成的ES6规范提供了大量的更新，目前主流的浏览器都已经支持了。标准委员会自此每年发布一个版本。

### 静态网页

当一个页面要显示内容，可以在div中嵌套内容，把具体显示的内容，链接，图片都以硬编码的方式写在html中，这是最简单直接的方式，也是最不灵活的。例如：

```html
<div class="media-content">
  <div class="content">
    <p>
    <strong>
      <a href="#" class="has-text-info">Yellow Pail</a>
      <span class="tag is-small">#4</span>
    </strong>
    <br>
      On-demand sand castle construction expertise.
    <br>
    <small class="is-size-7">
      Submitted by:
      <img class="image is-24x24" src="../public/images/avatars/daniel.jpg">
    </small>
    </p>
  </div>
</div>
```

### 响应式界面

通过数据驱动界面的显示，当数据变化了view就可以动态更新显示内容和效果。

### 数据模型

简单模拟数据模型，把数据定义在一个js对象中，例如`Seed.js`文件中定义了一个投票数据列表，有了这个submissions数据对象，在页面上就可以直接使用submissions[i]来访问每一个数据

```javascript
window.Seed = (function () {
  const submissions = [
    {
      id: 1,
      title: 'Yellow Pail',
      description: 'On-demand sand castle construction expertise.',
      url: '#',
      votes: 16,
      avatar: '../public/images/avatars/daniel.jpg',
      submissionImage: '../public/images/submissions/image-yellow.png',
    },
    {
      id: 2,
      title: 'Supermajority: The Fantasy Congress League',
      description: 'Earn points when your favorite politicians pass legislation.',
      url: '#',
      votes: 11,
      avatar: '../public/images/avatars/kristy.png',
      submissionImage: '../public/images/submissions/image-rose.png',
    },
    {
      id: 3,
      title: 'Tinfoild: Tailored tinfoil hats',
      description: 'We have your measurements and shipping address.',
      url: '#',
      votes: 17,
      avatar: '../public/images/avatars/veronika.jpg',
      submissionImage: '../public/images/submissions/image-steel.png',
    },
    {
      id: 4,
      title: 'Haught or Naught',
      description: 'High-minded or absent-minded? You decide.',
      url: '#',
      votes: 9,
      avatar: '../public/images/avatars/molly.png',
      submissionImage: '../public/images/submissions/image-aqua.png',
    }
  ];

  return { submissions: submissions };
}());
```

### 应用程序实例

应用程序是vue应用的入口点，一个应用程序实例接受一个options对象，这个对象描述了这个实例的模版(template)，数据(data)，方法(methods)等属性。根应用程序实例可以和一个DOM元素绑定(mount)，这个DOM元素就是它的容器。要创建一个vue的应用实例，得先在页面中引入vue.js和相关的应用js代码。

#### 引入vue.js

`<script src="https://unpkg.com/vue"></script>`html中的这句引入了最新版本的vue.js。

`<script src="./main.js"></script>`这句引入了应用程序的主要代码

```html
<!DOCTYPE html>
<html>
<head>
  <link rel="stylesheet"
    href="https://cdn.bootcdn.net/ajax/libs/bulma/0.5.3/css/bulma.css">
  <link rel="stylesheet"
    href="https://cdn.bootcdn.net/ajax/libs/font-awesome/5.1.0/css/all.css">
  <link rel="stylesheet"
    href="../public/styles.css" />
</head>

<body>  
  <div id="app">
    <h2 class="title has-text-centered dividing-header">UpVote!</h2>
    <div class="section">
      <article 
        v-for="submission in sortedSubmissions" 
        v-bind:key="submission.id"
        class="media"
        v-bind:class="{ 'blue-border': submission.votes>=20}"
        >
        <submission-component
          v-bind:submission="submission"
          v-bind:submissions="sortedSubmissions">
        </submission-component>
      </article>
    </div>
  </div>
  <script src="https://unpkg.com/vue"></script>
  <script src="./seed.js"></script>
  <script src="./main.js"></script>
</body>
</html>
```

#### 创建应用

在main.js中可以创建一个应用实例或者称为根组件

```javascript
const upvoteApp = {
  data() {
    return {
      submissions: Seed.submissions
    };
  },
  computed: {
    sortedSubmissions() {
      return this.submissions.sort((a, b) => {
        return b.votes - a.votes;
      });
    },
  },
  
  components: {
    "submission-component": submissionComponent,
  },
};
// 创建一个应用实例，它绑定在id是app的div上
const app = Vue.createApp(upvoteApp).mount("#app");
```

这里应用绑定在id名称为app的div元素上，这个div元素作为应用的容器，其中可以使用应用的数据，方法和模板。

其中upvoteApp就是应用程序根组件的options对象，这里目前定义了应用的三个属性：

* data 用来返回这个组件的数据对象，例如submissions变量返回了Seed.js中的Seed.submissions对象，在vue的表达式中就可以使用这个submissions变量。
* computed 标识这个组件计算属性，只要计算函数中的数据变化，计算就会发生，而页面中可以像使用数据变量一样使用计算对象
* components 应用程序或根组件中可以定义它里面的子组件，子组件的名称为`submission-component`，它的options对象为`submissionComponent`，通过props可以把数据传递给子组件。

#### 使用数据

* 对于html标签的属性，可以使用**v-bind来动态绑定vue程序的数据**，例如超链接的href就可以直接使用data中的submissions对象
* 标签的内容可以使用`{{表达式}}`模板来使用数据变量，而这个语法可以和后端服务结合生成不同的模板

```html
<div id="app">
    <h2 class="title has-text-centered dividing-header">UpVote!</h2>
    <div class="section">
      <article class="media">
        <figure class="media-left">
          <img class="image is-64x64" v-bind:src="submissions[0].submissionImage">
        </figure>
        <div class="media-content">
          <div class="content">
            <p>
              <strong>
                <a v-bind:href="submissions[0].url" class="has-text-info">
                  {{submission[0].title}}
                </a>
                <span class="tag is-small">#{{submissions[0].id}}</span>
              </strong>
              <br>
                {{submissions[0].description}}
              <br>
              <small class="is-size-7">
                Submitted by:
                <img class="image is-24x24" v-bind:src="submissions[0].avatar">
              </small>
            </p>
          </div>
        </div>
        <div class="media-right">
          <span class="icon is-small" v-on:click="upvote(submissions[0].id)">
            <i class="fa fa-chevron-up"></i>
            <strong class="has-text-info">{{submissions[0].votes}}</strong>
          </span>
        </div>
      </article>
    </div>
  </div>
```

#### 列表渲染

对于数据列表，对每一个数据都手动写html代码太繁琐了，通过使用v-for语法可以遍历数据列表中的每一个元素，通过指定唯一key来让vue使用列表的每一个对象来创建子内容。下面例子中，article标签中使用了v-for语法，所以article标签会被按列表元素重复创建，key为每一个元素的唯一标识id，和其他语言中的for each语法一样，submission是列表sortedSubmissions中的每一个元素的代称，在下面就可以使用submission遍历每一个列表项了，而不用索引。同时根据数据有多少个，就会创建多少个article，而且可以根据数据的不同每个article可以有自己的动态设置。

` v-bind:class="{ 'blue-border': submission.votes>=20}"`表示当一个submission对象的votes变量的值大于20后，给article增加一个样式'blue-border'，即蓝色边框

```html
<article 
        v-for="submission in sortedSubmissions" 
        v-bind:key="submission.id"
        class="media"
        v-bind:class="{ 'blue-border': submission.votes>=20}"
        >
        <figure class="media-left">
          <img class="image is-64x64" v-bind:src="submission.submissionImage">
        </figure>
        <div class="media-content">
          <div class="content">
            <p>
              <strong>
                <a v-bind:href="submission.url" class="has-text-info">
                  {{submission.title}}
                </a>
                <span class="tag is-small">#{{submission.id}}</span>
              </strong>
              <br>
                {{submission.description}}
              <br>
              <small class="is-size-7">
                Submitted by:
                <img class="image is-24x24" v-bind:src="submission.avatar">
              </small>
            </p>
          </div>
        </div>       
      </article>
    </div>
  </div>
```

#### 处理事件

通过`v-on:`给一个标签增加事件处理，类似原生的JavaScript的事件处理函数，只是这里可以使用vue实例对象或组件的方法作为事件处理函数。

`<span class="icon is-small" v-on:click="upvote(submissions[0].id)">`就给这个span的内容绑定了一个click事件，当点击后，会调用vue组件的`upvote()`方法，并以一个submission对象的id作为参数，这样处理函数内就知道点击了列表中的哪一个。

#### 组件

随着开发功能模块 越来越多，方便相同的代码复用，例如一个数据的列表显示在多个功能的列表显示中都会用到，就可以把数据列表显示作为一个组件。根组件下面可以使用多个子组件。

```javascript
const submissionComponent = {
  template: 
  ` <div style="display: flex; width: 100%">
  <figure class="media-left">
  <img class="image is-64x64" v-bind:src="submission.submissionImage">
</figure>
<div class="media-content">
  <div class="content">
    <p>
      <strong>
        <a v-bind:href="submission.url" class="has-text-info">
          {{submission.title}}
        </a>
        <span class="tag is-small">#{{submission.id}}</span>
      </strong>
      <br>
        {{submission.description}}
      <br>
      <small class="is-size-7">
        Submitted by:
        <img class="image is-24x24" v-bind:src="submission.avatar">
      </small>
    </p>
  </div>
</div>
<div class="media-right">
  <span class="icon is-small" v-on:click="upvote(submission.id)">
    <i class="fa fa-chevron-up"></i>
    <strong class="has-text-info">{{submission.votes}}</strong>
  </span>
</div>
</div>
  `,
  props:['submission', 'submissions'],
  methods: {
    upvote(submissionId) {
      const submission = this.submissions.find(
        (submission) => submission.id == submissionId
      );
      submission.votes++;
    }
  },
};
```

