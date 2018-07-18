---
title: 封装好你的弹窗组件
---

弹窗组件、loading组件、提示框组件这些其实都可以直接写好组件，然后在每个页面通过import进来，使用数据流驱动的方式去显示或者隐藏，这样的方法好处就是弹出或者隐藏的效果可以做的很酷炫，毕竟可以使用<transition>包裹嘛，同时还可以很方便的使用css3修饰。

但是每个页面都重复import操作，就很累赘很不程序员了。

那么抽离这样的共用组件是很有必要的。

今天帮也是抽了点时间尝试了一下vue的组件抽离，发现也是很简单的。

<!--more-->

## 模板vue文件

首先我们先写好自己的组件模板popup.vue

``` bash
<template>
  <div class="popup p-f">
    <div class="popup-warp p-a">
      <h1 class="title t-a-c p-a" v-if="title">{{title}}</h1>
      <div class="content t-a-c" v-html="content"></div>
      <div class="button-wrap p-a t-a-c" v-if="!type">
        <button class="d-i-b" @click="handle">确定</button>
      </div>
      <div class="button-wrap p-a t-a-c" v-else-if="type == '2'">
        <button class="d-i-b has-mar" @click="handle">确定</button>
        <button class="d-i-b" @click="closeHandle">取消</button>
      </div>
    </div>
  </div>
</template>
<script>
 export default {
    data() {
      return {
        title: '',
        type: '',
        content: '',
        callback: null
      }
    },
    methods: {
      handle() {
        if (this.callback) this.callback();
        this.close();
      },
      closeHandle() {
        this.close();
      }
    }
  }
</script>
```

这里就不写css了。大家都懂css，写上去就碍大家阅读了。

一个弹窗组件，弹窗内容是需要动态设置的。那么，就需要一个js去统筹、关联这个模板了，

## index.js文件

``` bash

import Vue from 'vue'
import Popup from './index.vue'

let myPopup = {
  component: null,
  alert(options) {

    // 如果传入的是ref的dom对象。copy一份dom
    if (options.content.attributes) {
      let copyDom = options.content.outerHTML
      copyDom = copyDom.replace('popup-content', '');
      options.content = copyDom;
    }
    // 使用基础 Vue 构造器，创建一个“子类”。参数是一个包含组件选项的对象，这个选项对象最终会直接保存在“子类的”data属性下。
    const oVue = Vue.extend(Popup)
    myPopup.component = new oVue({
      data: options
    }).$mount()
    document.querySelector('body').appendChild(myPopup.component.$el)
  },
  close() {
    document.querySelector('body').removeChild(myPopup.component.$el)
  }
}
export default myPopup

```

通过这个js文件export的myPopup方法调用alert或者close就可以显示或者隐藏组件弹窗了。

这里有几个点要说明一下。

``` bash
if (options.content.attributes) {
  let copyDom = options.content.outerHTML
  copyDom = copyDom.replace('popup-content', '');
  options.content = copyDom;
}
```

这里主要是处理一些特殊的弹窗内容，比如当传入一个vue的dom的时候，这里的处理就是直接copy一份dom，并把里面的nodetext赋值给content，content是给弹窗组件作为渲染弹窗内容的。
<div class="tip">
<p>为什么要copy一份而且要replace('popup-content', '')呢？</p>
<p>因为我遇过这样的一个需求。用户点击领取奖品后，弹窗的内容除了领取成功文字，还会有最近热门的礼包介绍。热门礼包的介绍包括图片、文字描述等。那么这一块最好就是写成一个html结构，通过弹窗方法通过content传进去作为弹窗内容显示出来，但是这个html结构在前端页面肯定是不能给用户看到的，只有弹窗的时候才显示在弹窗里面，所以popup-content这个class就起作用了.</p>

<p>这个class就只有两行代码,opcity:0;display:none;</p>

<p>这下明白为什么我要copy一份dom并去除这个className了吧？</p>
</div>

[Vue.extend](https://vuejs.org/v2/api/#Vue-extend)，不了解这个方法的而已到这里看一下。

到了这里我们可以直接imort这个index.js进来并调取alert方法进行弹窗。

但是我们这样做了不也是全部页面import一次？这明显不是我们想要的。

我们需要的是直接 this.alert()或者this.close()。

也就是挂载到Vue的实例的prototype上。

## 挂载全局

因为我是在nuxt下写这个demo的，所以我这里直接是在plugin上新建一个common.js专门处理这些全局工具类或者方法。

上代码

``` bash
import Vue from 'vue'
import Popup from '../components/common/tool/popup/index.js'

const alertTool = {
  install(Vue) {
    // Vue.component('popup', Popup.alert)
    // 挂载vue实例
    Vue.prototype.alert = Popup.alert;
    Vue.prototype.close = Popup.close;
    .......
    # 你可以在这里继续给Vue.prototype挂载更多实例共享的方法
  }
}

Vue.use(alertTool)

```

这里的操作就很简单了，单纯是引入你写的js文件，并把文件内export的方法挂载全局。

## 最后

在nuxt.connfig.js的plugin里引用一次即可。

