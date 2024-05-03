---
title: Vue3 单文件组件
date: 2024-05-03 19:58:49
categories:
- programming
tags:
- web
- vue
- frontend
---

## Vue 3单文件组件

### 创建工程

从官方[快速上手](https://cn.vuejs.org/guide/quick-start.html)为例创建工程

1.  `npm create vue@latest `使用官方的创建工具按步骤创建一个web应用，默认使用的vite作为构建工具
2. ` npm install `安装依赖
3. ` npm run dev `运行工程，生产环境使用` npm run build `

### 工程目录

* node_modules 当前应用程序通过`npm install`安装的依赖库

* package.json 列出了本地安装的npm包，以及其中的**scripts**部分列出了当前应用可以执行的npm命令；devDependencies是仅在开发阶段使用的依赖包，例如一些vue的插件；dependencies是开发和发布后都依赖的包。

* package-lock.json 记录了当前应用程序编译的依赖库的版本

* public 应用使用的第三方公共资源例如图标，字体，样式

* index.html 应用程序的根页面，其中引入依赖的外部样式表依赖，以及Vue实例mount的DOM元素也在这个页面中

* src JavaScript代码，其中`main.js`里面定义了应用的入口点，并把`App.vue`文件定义根`App`组件引入进来

  ```javascript
  import { createApp } from 'vue'
  import App from './App.vue'
  
  createApp(App).mount('#app')
  ```

Vue CLI提供了应用程序标准Webpack 配置，它使用`webpack`和`webpack-dev-server`，为我们的应用提供了编译，lint，测试和运行服务。

### 单文件组件

vue提供了单文件组件方式用来编写一个组件。这样一个组件的所有内容放在一个`.vue`文件中。它一般包括3个部分：

* template 这个组件的html标记内容
* script 组件的逻辑js代码，声明组件中的对象
* style 组件使用的样式

Webpack这样的构建工具可以把vue组件文件编译成普通的JavaScript模块，从而可以在浏览器中执行。

### 组件数据管理

应用的运转需要组件之间数据传递。根据组件之间的关系，有不同的数据通信方式。

#### 父->子组件

子组件不能直接访问父组件中对象。需要使用**props**让父组件的数据传递给子组件，这种方式可以清晰表达组件之间的数据流。

#### 子->父组件

子组件使用**自定义事件**与父组件通信。vue中通过在一个组件中`$emit(nameOfEvent)`发出事件，再另一个组件中监听事件`$on(nameOfEvent)`,通过事件可以传递数据。

#### 同级组件之间

同级组件之间使用三种方式传递数据：

* 全局event bus
* 简单共享存储对象
* 状态管理库Vuex

##### Global Event Bus

使用应用全局的自定义事件可以简单的在所有的组件之间传递数据。这种方法不推荐，对应用的状态管理太乱。

##### Vuex

显示的定义getter, mutations, actions的状态对象基础上的库

#### 简单状态管理

状态简单理解为数据，状态管理也就是应用程序级别的数据管理。

通过仓库(store)模式来实现在多个组件之间共享数据。仓库管理状态的行为，变化等。所有对仓库中数据的更改行为都需要在仓库中定义，用来确保集中管理应用的状态。

例如下面定义了一个仓库中有一个state，里面有一个数字列表，通过 `pushNewNumber(newNumberString)`方法可以给数字列表增加数字，这个更改方法就定义在仓库里面，其他组件可以调用这个方法。当一个组件调用仓库的`pushNewNumber`**mutation**来修改**状态**后，状态的变化会触发另一个使用store中**状态**的组件更新视图view。

```javascript
export const store = {
    state: {
        numbers: [1, 2, 3]
    },    
    
    pushNewNumber(newNumberString) {
        this.state.numbers.push(Number(newNumberString));
    }
}
```

一个组件可以访问store中的方法来修改状态

```html
<template>
    <div>
        <input v-model="newNumber" type="number" />
        <button @click="pushNewNumber(newNumber)">Add new number</button>
    </div>
</template>

<script>
import { store } from './store.js';
export default {
    name: 'NumberSubmit',
    data () {
        return {
            newNumber: 0
        },
    }
    methods: {
        pushNewNumber(newNumber) {
            store.pushNewNumber(newNumber);
        }
    }
}
</script>
```

### 创建一个vue应用的基本步骤

1. 创建一个静态版本的app
2. 把这个app分解为多个组件
3. 使用父->子的数据流来初始化状态传递
4. 创建状态变化Mutation和组件派发dispatchers