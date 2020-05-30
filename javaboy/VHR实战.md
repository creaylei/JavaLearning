# VHR实战

> 2020-0202  由于前面的内容大部分都自己学过，这里先直接进入VHR实战的部分

涉及知识点

后端：

1. Spring Boot(SSM)
2. Spring Security
3. Redis
4. POI/邮件发送/消息中间件
5. Mysql

前端：

1. Vue
2. Axios
3. ElementUI
4. Vuex



## Vue简介

### 1.MVVM

Model-View——View-Model：是一个数据显示-显示数据双向绑定的设计理念。

### 2. 开始Demo

### 1. 新建Vue对象

```javascript
var app = new Vue({
	el: '#app',        //这里是选中的id
	data: {
		message: 'hello world'       //变量的内容
	}
})
```



### 2. 动态绑定

**V-bind： 绑定变量** 

指令前面 v- 表示Vue提供的特殊的行为。

```javascript
<div id="app" v-bind:title="title">
    {{ message }}
</div>
```

**V-if / V-show**

```js
<div v-if = 'flag'>Hello Vue!</if>
```

这里如果是false，就会把当前div移除

```js
<div v-show="flag">hello javaboy!</div>
```

这里如果flag = false，将用样式将该div隐藏，这时候浏览器检查可以看到

```javascript
<div id="app"><div style="display: none;">hello javaboy!</div></div>
```

注意这里的  display:none,   v-show通过该字段将div隐藏起来

**V-for = "i in num"**

```javascript
<div id="app">
    <table border="1">
        <tr v-for="i in num">
            <td v-for="j in i">{{j}}x{{i}}={{j*i}}</td>
        </tr>
    </table>
</div>
<script>
    var app = new Vue({
        el: '#app',
        data: {
            num: 9
        }
    })
</script>
```

上面这里是个9*9乘法表

**函数**

```js
var app = new Vue({
        el: "#app",
        data: {
            message: "Hello world!"
        },
        methods: {                      着重是这里
            reverseMethod() {
                this.message = this.message.split("").reverse().join("");
            }
        }
    })
```

方法methods和data是同级的

**v-model**

```
<input v-model="message">
```

这个标签是绑定变量和显示，如果message变量发生变化，那么

**Vue组件：Vue.component**

Vue可以自定义重复组件（**组件相当于一个标签**），然后构成复杂页面

**Vue.component('变量名称',  {template: " " })**

```javascript
// 定义名为 todo-item 的新组件
Vue.component('todo-item', {
  template: '<li>这是个待办项</li>'
})

//添加变量
Vue.component('todo-item', {
  props: ["name"]
  template: '<li>这是个待办项 {{ name }}</li>'
})
```

用法

```javascript
<todo-item name='zhanglei'></todo-item>
```

### 3. Vue实例

1. 数据属性，Vue是MVVM的，即改动数据或者视图，是联动的。但是

`Object.freeze()` 可以阻止后期的改动。



2. 实例属性 $

   ```
   var data = { a: 1 }
   var vm = new Vue({
     el: '#example',
     data: data
   })
   
   vm.$data === data // => true
   vm.$el === document.getElementById('example') // => true
   ```

3. 实例生命周期**钩子** (特定关键词)

   `created`、`mounted`、`updated`、`destroyed`，特定时期可以做一些处理

   ```vue
   new Vue({
     data: {
       a: 1
     },
     created: function () {
       // `this` 指向 vm 实例
       console.log('a is: ' + this.a)
     }
   })
   // => "a is: 1"
   ```



### 4. 模板语法

插值：

```
1. <div>{{ message }}</div>      html标签外，可以直接用 {{}}来插入变量
2. html标签里面， 
	 v-html:"变量名"        将变量映射为html链接
	 <span v-html="message"></span>
	 
	 Vue里面， data{ message: "www.baidu.com" }
	 
3. v-bind： 绑定变量的值
   <button v-bind:disabled="isButtonDisabled">Button</button>
   
   
4. 指令  v-if,   
```

缩写

| 完整写法 | 缩写   |
| -------- | ------ |
| v-bind   | :      |
| v-on     | @click |
|          |        |



### 5. 计算属性和侦听器

复杂逻辑用计算属性，模板内的表达式维护只用来显示就好

#### 计算属性 computed

```js
<div id="example">
  <p>Original message: "{{ message }}"</p>
  <p>Computed reversed message: "{{ reversedMessage }}"</p>
</div>

var vm = new Vue({
  el: '#example',
  data: {
    message: 'Hello'
  },
  computed: {
    // 计算属性的 getter
    reversedMessage: function () {
      // `this` 指向 vm 实例
      return this.message.split('').reverse().join('')
    }
  }
})
```

#### 方法	methods

```js
<p>Reversed message: "{{ reversedMessage() }}"</p>

// 在组件中
methods: {
  reversedMessage: function () {
    return this.message.split('').reverse().join('')
  }
}
```

> 两种方式都可以实现一个方法，但是**计算属性**是具有**缓存**，可以再重复调用的时候，直接返回值。
>
> 方法调用，每次都会重新计算。

#### 侦听器  watch

看个例子

```js
var app = new Vue({
        el: '#app',
        data: {
            firstName: "san",
            lastName: "zhang",
            fullName:"zhang san"
        },
        watch: {
            firstName(val) {
                this.fullName = this.lastName + " " + val;
            },
            lastName(val) {
                this.fullName = val +" " +this.firstName;
            }
       
```

这个例子并不恰当，但是可以参考。当开发者工具中 firstName 或者 lastName发生变化，显示也会发生变化。

适用于： **异步操作、耗时操作**  ， computed是有缓存，平常用的更多一些

> 微人事中树的搜索使用

### 6 Class 与 Style 绑定

#### 绑定html的class

```js
<div id="app">
    <div v-bind:class="{mydiv:flag}">{{msg}}</div>
</div>
<script>
    var app = new Vue({
        el: '#app',
        data: {
            msg: "hello javaboy",
            flag: true
        }
    })
</script>
<style>
    .mydiv {
        color: orange;
        font-weight: bold;
    }
</style>
```

这里主要学习 `<div v-bind:class="{mydiv:flag}">{{msg}}</div> ` 中绑定class的写法。 多个class可以并存，这时候就可以定义一个对象。

#### 绑定html的style

```js
<div id="app">
    <div v-bind:style="{color:fontColor,fontSize:fontSize+'px'}">{{msg}}</div>
    <div v-bind:style="styleObj">{{msg}}</div>         //对象的引用方式
</div>
<script>
    var app = new Vue({
        el: '#app',
        data: {
            msg: "hello javaboy",
            fontSize: 30,
            fontColor: "red",
            styleObj: {              //定义成对象
                color: "orange",
                fontSize: "50px"
            }
        }
    })
</script>
```

### 7 条件渲染

#### `V-if `  (偶尔显示隐藏)

连用  v-if  v-else

v-if   v-else-if   v-else-if

```js
<div v-if="type === 'A'">
  A
</div>
<div v-else-if="type === 'B'">
  B
</div>
<div v-else-if="type === 'C'">
  C
</div>
<div v-else>
  Not A/B/C
</div>
```



key元素，复用元素，如果不复用，加key

---

#### `V-show`  （频繁的显示隐藏）

v-show只是切换css里面的 display

不支持template、v-else

> `v-if` 是“真正”的条件渲染，因为它会确保在切换过程中条件块内的事件监听器和子组件适当地被销毁和重建。
>
> `v-if` 也是**惰性的**：如果在初始渲染时条件为假，则什么也不做——直到条件第一次变为真时，才会开始渲染条件块。
>
> 相比之下，`v-show` 就简单得多——不管初始条件是什么，元素总是会被渲染，并且只是简单地基于 CSS 进行切换。
>
> 一般来说，`v-if` 有更高的切换开销，而 `v-show` 有更高的初始渲染开销。因此，如果需要非常频繁地切换，则使用 `v-show` 较好；如果在运行时条件很少改变，则使用 `v-if` 较好。

### 8.列表渲染

#### `V-for`

`v-for`遍历数组

```js
<div v-for="item in items"></div>

var example2 = new Vue({
  el: '#example-2',
  data: {
    parentMessage: 'Parent',
    items: [
      { message: 'Foo' },
      { message: 'Bar' }
    ]
  }
})
```

其中，in也可以换位of。

`v-for`遍历对象

```html
<ul id="v-for-object" class="demo">
  <li v-for="value in object">
    {{ value }}
  </li>
</ul>
```

```js
new Vue({
  el: '#v-for-object',
  data: {
    object: {
      title: 'How to do lists in Vue',
      author: 'Jane Doe',
      publishedAt: '2016-04-10'
    }
  }
})
```

> 也可以提供第二个的参数为 property 名称 (也就是键名)：
>
> 以及第三个参数，index

```html
<div v-for="(value, name) in object">
  {{ name }}: {{ value }}
</div>
```

注意，不同的元素最好加上一个key，来标记不同的元素

数组更新检测

> 变异方法
>
> - push()`
> - `pop()`
> - `shift()`
> - `unshift()`
> - `splice()`
> - `sort()`
> - `reverse()`
>
> 非变异方法
>
> `filter()`、`concat()` 和 `slice()` 。它们不会改变原始数组，总返回一个新数组

### 9. 事件处理

#### V-on

可以用 `v-on` 指令监听 DOM 事件，并在触发时运行一些 JavaScript 代码。

```js
<div id="app">
    <div>{{counter}}</div>
    <input type="button" value="自增" v-on:click="counter++">
    <input type="button" value="自减" v-on:click="counter--">
</div>
<script>
    var app = new Vue({
        el: '#app',
        data: {
            counter:0
        }
    })
</script>
```

### 10. 表单输入绑定

#### `v-model` 

> 可以使用v-model指令在` <input>  <textarea> <select> ` 元素上创建双向绑定数据

```html
<input v-model="message" placeholder="edit me">        文本

<textarea v-model="message" placeholder="add multiple lines"></textarea>   多行文本


<input type="checkbox" id="checkbox" v-model="checked">     复选框 checkbox

<input type="radio" id="one" value="One" v-model="picked">    单选radio  value是选中的值


下拉框
<select v-model="selected">   //其中加一个 multiple 就是多选 
        <option disabled value=" ">请选择</option>
        <option>地球</option>
        <option>月球</option>
        <option>火星</option>
    </select>
```



#### 修饰符  v-model.lazy.number等

.lazy    转变为change事件进行同步。     即光标点到输入框之外进行同步

.number    将用户的输入转换为  数字类型

.trim    取消输入前后的空白

### 11. 组件基础

#### Vue.component

组件里面，data必须定义为一个函数

```js
data: function () {
  return {
    count: 0
  }
}
```

如果还像之前那样直接声明，当其他部分使用同一个组件的时候，会受到影响。

一个完整的例子：

```html
<div id="app">
    <creaylei></creaylei>
</div>
<script>
    Vue.component("creaylei", {
        data() {
            return {
                counter:0
            }
        },
        template: '<button v-on:click="counter++">hello {{counter}}~</button>'
    })
    var app = new Vue({
        el: '#app',
        data: {

        }
    })
</script>
```

#### 通过 Prop 向子组件传递数据

```html
Vue.component('blog-post', {
  props: ['title'],
  template: '<h3>{{ title }}</h3>'
})


new Vue({
  el: '#blog-post-demo',
  data: {
    posts: [
      { id: 1, title: 'My journey with Vue' },
      { id: 2, title: 'Blogging with Vue' },
      { id: 3, title: 'Why Vue is so fun' }
    ]
  }
})

//渲染的时候
<blog-post
  v-for="post in posts"
  v-bind:key="post.id"
  v-bind:title="post.title"
></blog-post>
```

注意，所有的定义都必须放到一个标签中去，否则组件不能被识别

#### 插槽

`<slot></slot>`  在 component中的 template中，  加入这个标签，然后在使用组件的地方，可以加一个标签，替代slot的位置。也可以取名字



## SPA

Single Page Application  单页面应用

只写一个html页面，通过js来操作，修改当前html页面，类似于跳转。

优劣：

> 优：擅长做后台管理系统
>
> 劣：没法进行搜索引擎优化，后台管理系统不需要搜索引擎优化。



### Vue如何创建SPA应用

vue-cli2

Vue-cli3(微人事以这个为准)

#### 起步

安装 node.js 和 npm， node.js里面包含了npm，npm可以类似对标为后端的maven

**创建一个vue项目**

```
vue init webpack project
```

![image-20200331163220475](https://tva1.sinaimg.cn/large/00831rSTly1gdd70y3wvgj315c0bcqbt.jpg)

然后 `npm run dev`  就可以启动了，但是有时候会报错

![image-20200331163347221](https://tva1.sinaimg.cn/large/00831rSTly1gdd72f36qfj30zs0d8aiz.jpg)

> 这是因为，localhost和127.0.0.1 没有绑定。
>
>  mac下的解决办法是  `sudo vim /etc/hosts`   加入  127.0.0.1 localhost   或者用switchhost

#### 项目目录介绍

![image-20200331164121188](https://tva1.sinaimg.cn/large/00831rSTly1gdd8kogliyj30ee0jiq3s.jpg)

build： 是打包相关的

config：配置相关

node_modules：本地启动依赖包，发布的时候不会用到，几个G大，不会带到生产去

src：后面开发相关的都在这里

![image-20200403111731614](https://tva1.sinaimg.cn/large/00831rSTly1gdgesazkabj31le0ggjvh.jpg)

> 这里 main.js 相当于后端的main函数，启动函数，项目入口函数
>
> 从main进去，
>
> el是指渲染的div的id,  
>
> router是路由(即路径和组件映射表)，main函数里是注册使用。  具体的映射路径，可以看包router下的index.js里，如下
>
> ```js
> Vue.use(Router)
> 
> export default new Router({
>   //这个就的路径映射
>   routes: [
>     {
>       path: '/',
>       name: 'HelloWorld',
>       component: HelloWorld
>     }, {
>       path: '/hello',
>       name: 'hello',
>       component: Hello
>     }
>   ]
> })
> ```
>
> component是组件，通常的写法是  key:value。 
>
> template就是使用的组件名称。

