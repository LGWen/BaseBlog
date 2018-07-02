---
title: Vue中响应式检测数组、对象的属性变化（增加或者删除）
---
Vue的官方文档说到： [Vue 最独特的特性之一，是其非侵入性的响应式系统。](https://cn.vuejs.org/v2/guide/reactivity.html) 数据模型data仅仅是普通的 JavaScript 对象。而当你修改它们时，视图会进行更新。

但是有时候发现修改了model层数据模型data的时候，view视图层却不能及时、有效的得到更新。往往这类的问题出现都是因为动态操作了数组或者对象字面量的属性导致。

这里会有几种情况例子，但是其实都是<code>同一个问题</code>。

<!--more-->

## 问题

### 一 ： <code>Vue并不能检测已经创建的实例上动态添加新的根级响应式属性</code> 

所谓的根级就是data下。

``` bash
var vm = new Vue({
  data:{
  a:1
  }
})

// `vm.a` 是响应的

vm.b = 2
// `vm.b` 是非响应的
```

<div class="tip">
   官方的解释是：受现代 JavaScript 的限制 (以及废弃 Object.observe)。由于 Vue 会在初始化实例时对属性执行 getter/setter 转化过程，所以属性必须在 data 对象上存在才能让 Vue 转换它，这样才能让它是响应的
</div>

### 二 ： <code>Vue并不能检测到已对象属性中属性的添加或者删除</code>

在我们处理业务逻辑的时候，给data选项里的一个空数组或者空对象增加属性并赋值，然后我们页面是动态依赖这个对象或者数组的属性。
``` bash
   <li v-for="item,index in items" v-on:click="handle(index)">
        <span>{{item.name}} is :</span>
        <span>{{numbers[index]}}</span>
    </li>

    data(){
        return {
            numbers: [],
            items: [
                {name: 'jjj'},
                {name: 'kkk'},
                {name: 'lll'},
            ]
        }
    }

    methods: {
        handle: function (index) {
            // WHY: 更新数据，view层未渲染，但通过console这个数组可以发现数据确实更新了
            if (typeof(this.numbers[index]) === "undefined" ) {
                this.numbers[index] = 1;
            } else {
                this.numbers[index]++;
            }
        }
    }

```
结果发现：点击之后数字并没有在view层更新


### 三： <code>当你利用对象的索引直接设置一个项时</code>

``` bash
   <li v-for="item,index in items" v-on:click="handle(index)">
        <span>{{item.name}} is :</span>
        <span>{{numbers[index]}}</span>
    </li>

    data(){
        return {
            numbers: [
                {
                name: "k"
                },
                {
                name: "kk"
                },
                {
                name: "kkk"
                }
            ],
            items: [
                {name: 'jjj'},
                {name: 'kkk'},
                {name: 'lll'},
            ]
        }
    }

    methods: {
        handle: function (index) {
            this.numbers[_index].name = _index;
        }
    }
```
结果发现：点击item并没有在view层更新index


### 四：  <code>当你修改数组的长度时</code>

``` bash
var vm = new Vue({
  data: {
    items: ['a', 'b', 'c']
  }
})
vm.items[1] = 'x' // 不是响应性的
vm.items.length = 2 // 不是响应性的

```




## 原因

要解释上述问题，最好的方法就是了解一下Vue在render数据的时候，是如何实现数据的双向绑定的。

Vue实例的data选项，是一个普通的JavaScript对象。

Vue在render前，将会递归遍历data选项上所有的属性，并使用[Object.defineProperty](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)把这些属性全部转化为<code>setter/getter</code>。如果想要获取这个属性的内部值，Vue会内部调用getter进行取值，同理赋值的时候，Vue内部就会调用setter进行设值。
<div class="tip">
    但是Object.defineProperty不支持IE8，所以Vue就不兼容IE8以及更低版本的浏览器了。
</div>

那么，到这里就可以解释到Vue是怎么进行数据的双向绑定了。

每个组件实例都有自己对应的watcher实例对象，它会在组件渲染的过程中把属性记录为依赖，当依赖项的setter被调用时（也就是data选项里的属性被修改操作，一般为赋值），会通知watcher重新计算。从而致使它关联的组件进行更新。

<div class="tip">
    补充一点setter和getter的知识：
    访问器属性不包含数据值，他们包含一对getter函数和setter函数（这两个函数不是必须的）。在读取访问器属性时，会调用getter函数，这个函数负责返回有效的值；在写入访问器属性是，会调用setter函数并传入新值，这个函数负责决定如何处理数据。
</div>

也就是在渲染的过程中，<code>data作为页面的基础数据进行渲染。这个时候是首先是递归调用该对象属性的getter函数进行取值，然后getter函数的每一次调用都会通知watcher，告知属性将被调用（getter）并让watcher将之声明为依赖。如果属性被操作（setter进行设值）并发生变化，setter函数的每一次调用也会通知watcher对象，watcher对象收到通知就会重新渲染组件，以此来完成视图的更新，达到响应式。</code>


那么我们回到问题，为什么我们有时候操作对象属性的时候为什么view视图层不能及时、有效的得到更新。

因为Vue在初始化实例的时候会先递归遍历并关联data选项里面以有的属性，执行了setter/getter转化过程，所以属性必须开始就在对象上，这样才能让Vue转化它。 
至于后来对视图进行操作才给相应的空数组或对象进行增加属性操作的时候，Vue就不能检测到被操作为对象中属性的添加或者删除。所以也就达不到双向绑定了。



## 解决方法

### <code>使用 Vue.set(object, key, value) 方法将响应属性添加到嵌套的对象上。 还可以使用 vm.$set 实例方法，这也是全局 Vue.set 方法的别名。</code>

例如文章的例子一修改为：

``` bash
Vue.set(vm.someObject, 'b', 2) || this.$set(this.someObject,'b',2)

```

例子二是为已有对象上添加一些属性，对此这里会分两种情况。

### <code>情况一：操作数组，数组对固定的索引设值，使用 Vue.set(object, key, value)</code>

``` bash
    methods: {
        handle: function (index) {
            // WHY: 更新数据，view层未渲染，但通过console这个数组可以发现数据确实更新了
           if (typeof(this.numbers[index]) === "undefined" ) {
             this.$set(this.numbers, index, 1);
           } else {
             this.$set(this.numbers, index, ++this.numbers[index]);
           }
        }
    }
```

### <code>情况二：操作数组，数组不针对固定的索引设值，可以使用push操作</code>

``` bash
    methods: {
        handle: function (index) {
            this.numbers.push(index)
        }
    }
```

### <code>操作数组或者操作对象字面量增加属性，使用 Object.assign()创建一个新的数组/对象，让它包含原对象的属性和新的属性：</code>

``` bash
    methods: {
        handle: function (index) {
           // 代替 this.$set(this.numbers, index, 1);
            this.numbers = Object.assign([], this.numbers, [index])
        }
    }
```

### <code>对于问题四，操作数组的长度。Vue 包含一组观察数组的变异方法，所以它们也将会触发视图更新。这些方法如下：</code>

``` bash
push()
pop()
shift()
unshift()
splice()
sort()
reverse()
```

以上这些其实官方文档都有写明，只是在看文档的时候并不能在脑海里想象得到一些使用用例，导致就算看过这一块文档介绍但是遇到这一问题还是没能找到原因所在。

[关于操作数组的官方文档参考：列表渲染](https://cn.vuejs.org/v2/guide/list.html#注意事项)

[关于响应式的官方文档参考](https://cn.vuejs.org/v2/guide/reactivity.html)

