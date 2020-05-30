# Vue入门

## 1. Mac安装vue开发环境

1 安装Nodejs, 使用brew安装

 `brew install nodejs`

2 获取nodejs模块安装目录访问权限

`sudo chmod -R 777 /usr/local/lib/node_modules/`

3 安装淘宝镜像，国内直接使用 npm 的官方镜像是非常慢的，所以这里使用淘宝 NPM 镜像

​	3.1 更改npm源:

​	`npm config set registry https://registry.npm.taobao.org` 

​	3.2 配置后可通过下面方式来验证是否成功，如果显示“[https://registry.npm.taobao.org](https://registry.npm.taobao.org/)”则说明配置成功

​	`npm config get registry` 

​	3.3 然后安装淘宝的cnpm

​	`npm install -g cnpm --registry=https://registry.npm.taobao.org`

安装成功展示

```javascript
MacBook-Pro:~ hu$ sudo npm install -g cnpm --registry=https://registry.npm.taobao.org
npm WARN deprecated socks@1.1.10: If using 2.x branch, please upgrade to at least 2.1.6 to avoid a serious bug with socket data flow and an import issue introduced in 2.1.0
/usr/local/bin/cnpm -> /usr/local/lib/node_modules/cnpm/bin/cnpm
+ cnpm@6.0.0
added 632 packages from 843 contributors in 22.264s
```

4 用淘宝的cnpm安装vue

```javascript
cnpm install vue
cnpm install --global vue-cli
```

这里环境就配置完成了

5 运行自己的vue文件。

> 一定要将127.0.0.1 映射为localhost, 才能开启成功

## 2. 目录结构

这里先看下src包里面的文件

src

- assets： 放置图片、logo等
- components： 组件文件，类似于一个个小的vue文件
- App.vue： 项目入口文件，可以直接将组件写在这里，而不用components，但是每个独立的模块最好还是分开的好

- Main.js： 项目核心文件

static: 静态资源

index.html  首页入口文件

package.json ： 项目配置文件



## 3. 基础语法

### 1. 模板语法

- 文本：`{{ }}`  双大括号

- html： `v-html` 输出 html代码

  ```html
  v-html
  <div v-html='myname'></div>   输出myname指向带有html标签的内容
  
  v-bind  ： 属性中的值
  <div v-bind:class="{'class1:use'}"> 指令 </div>  可以设置use为 true/fasle 来控制
  
  支持js表达式
  
  ```

- 指令：指带有 `v-`  前缀的特殊属性

  ```javascript
  <div id="app">
      <p v-if="seen">现在你看到我了</p>
  </div>
      
  <script>
  new Vue({
    el: '#app',
    data: {
      seen: true
    }
  })
  </script>
  ```

- 最常用 v-bind(绑定属性)   v-on(监听事件)

  ```javascript
  缩写
  <!-- 完整语法 -->
  <a v-bind:href="url"></a>
  <!-- 缩写 -->
  <a :href="url"></a>
  
  <!-- 完整语法 -->
  <a v-on:click="doSomething"></a>
  <!-- 缩写 -->
  <a @click="doSomething"></a>
  ```

### 2. 条件语句

v-if 和 v-else

```javascript
<div id="app">
    <div v-if="Math.random() > 0.5">
      Sorry
    </div>
    <div v-else>
      Not sorry
    </div>
</div>
```

V-else-if

```html
<div v-if="type === 'A'">
      A
    </div>
    <div v-else-if="type === 'B'">
      B
    </div>
```

### 3. 函数和计算

#### 3.1 computed: 计算属性vs方法

```javascript
new Vue({
  el: '#app',
  data: {
    message: 'Runoob!'
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

reverseMessage可以直接使用

#### methods

> 可以使用 methods 来替代 computed，效果上两个都是一样的,但是 computed 是基于它的依赖缓存，只有相关依赖发生改变时才会重新取值。而使用 methods ，在重新渲染的时候，函数总会重新调用执行。



#### 3.2 计算属性vs侦听器

侦听器 **watch**

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

### 4 Class 与 Style 绑定

#### 4.1 绑定html的class

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

看到1