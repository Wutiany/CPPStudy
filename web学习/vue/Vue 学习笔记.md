# Vue 学习笔记

**前端四要素**

* 逻辑
  * 判断
  * 循环
* 事件
  * 浏览器事件：`window`，`document`
  * DOM 事件：增、删、遍历、修改节点元素内容
  * `jQuery`，`Vue`，`React`（框架）
* 视图
  * `html`
  * `css`
  * `layui`（框架）
* 通信
  * `Ajax` 
    * `jQuery`封装
    * `axios`（vue）

  	 

# Vue 7 大属性

## el

* 相当于选择器，选择元素
* 用来指示 `vue` 编译器从什么时候开始解析 vue 语法，等于占位符

## data

* 可以操作的数据，相当于 `view` 中的内容
* `view` 的内容与 `vue model` 进行的双向绑定（实时更新）

## template

* 设置模板，用来替换页面元素，包括占位符

## methods

* **事件驱动**的一系列方法
* 即**业务逻辑**

## render

* 创建正真的**虚拟 DOM**

## computed

* 用来计算的

## watch

* `watch:function(new,old){}`
* 监听 `data` 中数据的变化
* 两个参数，一个返回新值，一个返回旧值

# Vue 语法

## v-bind

* 动态将数据绑定到 `html` 属性上
* v-bind:属性名="表达式"

## v-if - else

* `v-if="return true or false"`，字符串作为方法，可以是一个判断的表达式，也可以是一个结果
* `v-else`

## v-for

* 渲染**一组数据**
* v-for="item in itmes"

## v-on

* 绑定事件
* v-on:事件="事件处理函数"

## v-model

* 双向绑定，将视图**绑定模型的数据**，视图内容变，模型数据变
* v-model="message"

* 会**忽略**表单元素的 `value`、`checked`、`selected` 特性的**初始值**而**总是**使用 Vue **实例的数据**作为**数据来源**

# Vue 组件

**组件就相当于一个模板，作为标签来使用**

## 组件定义

* Vue.component("组件名", {参数列表});

  * props: 组件接受的数据
  * template: 组件的模板

* 使用： <组件名 v-for="item in items" v-bind="item"></组件名>

  * 使用的时候，将==属性==**绑定（v-bind）**实例中的**数据**，但是这些数据**不能直接**传给组件，组件对象需要一个**接收传递值（属性）**的**数组**（props），接收后在组件中进行使用

* 实际：

  * **使用**相当于 `view` 层，可以将属性绑定实例的数据，组件只能通过**属性**来获取绑定的数据
  * **组件定义**相当于 `model` 层，需要获取 `view` 的数据（v-bind & props）才能处理**视图层获取**的数据（属性）
  * `view` 层，`model` 层交互使用过**属性的传递**来做的


## slot

* 插槽相当于一个模板中增加了**插槽标签**，可以**动态绑定**其他模板，实现**动态拔插**
* 实际还是使用属性进行数据的交互

```html
// 插槽的使用
<div id="app">
    // 先放置一个带有插槽的模板
    <todo>
        // 将模板插入到对应的插槽 slot=插槽名，将 vue 实例中的数据通过属性传入到组件中
        <todo-title slot="todo-title" :title="todoTitle"></todo-title>
        <todo-items slot="todo-items" v-for="item in todoItems" :itme="item"></todo-items>
    </todo>
</div>


// 一个带有插槽的组件
// slot 需要绑定插槽的名，上面使用的时候，也需要绑定对应的插槽的名，才能对应
<script>
    Vue.component("todo", {
	template: '<div> \
        		<slot name="todo-title"></slot> \
        		<ul> \
                    <slot name="todo-items"></slot> \
        </ul> \
        </div>'
})

Vue.component("todo-title", {
	props: ['title'],
	template: '<div>{{title}}</div>'
})

Vue.component("todo-itmes", {
	props:['item'],
	template: '<li>{{item}}</li>'
})


// vue 实例，用以给视图提供数据等操作
var vm = new Vue({
 el: "#app",
data: {
    todoTitle: "",
    todoItems: []
}
})
</script>

```

## 自定义组件

* 组件内**不能直接**使用 `vue` 实例的方法

* 但是通过自定义事件：`v-on:remove="vue function"`， 将**实例化组件**中的事件传入组件中

  * 实际，相当于实例化的组件向组件中**传递了一个事件**（等同于传递属性一样）
  * 组件通过 `this.$emit('事件名', 参数)` 来捕获事件

  ```html
  <!DOCTYPE html>
  <html lang="en">
    <head>
      <title>Vite App</title>
    </head>
    <body>
  
      <!-- // 插槽的使用 -->
      <div id="app">
          <!-- // 先放置一个带有插槽的模板 -->
          <todo>
              <!-- // 将模板插入到对应的插槽 slot=插槽名，将 vue 实例中的数据通过属性传入到组件中 -->
              <todo-title slot="todo-title" :title="todoTitle"></todo-title>
              <!-- 传入 item 和 index 属性， 组件会获取到这个属性的值 -->
              <!-- 传入事件，让组件去捕获这个事件 -->
              <todo-items slot="todo-items" v-for="(item,index) in todoItems" :item="item" :index="index" @remove="removeItems(index)" ></todo-items>
          </todo>
      </div>
  
  
  
  
      <script src="https://unpkg.com/vue@2"></script>
      <script>
      Vue.component("todo", {
  	    template: '<div> \
          		<slot name="todo-title"></slot> \
          		<ul> \
                  <slot name="todo-items"></slot> \
                  </ul> \
              </div>'
          });
  
      Vue.component("todo-title", {
          props: ['title'],
          template: '<div>{{title}}</div>'
      });
  
      Vue.component("todo-items", {
          props:['item', 'index'],
          // 组件调用组件定义的方法，自定义的方法通过捕获实例化动态传进来的事件来进行操作
          template: '<li>{{item}}<button @click="removeLi">删除</button></li>',
          methods: {
              removeLi: function (index) {
                  // this.$emit 相当于去捕获了实例化组件传递的事件
                  this.$emit('remove', index);
              }
          }
      });
  
      var vm = new Vue({
          el: "#app",
          data: {
              todoTitle: "编程语言",
              todoItems: ["java", "python", "go"]
          },
          methods: {
              removeItems: function (index) {
                  this.todoItems.splice(index, 1);
              }
          }
      });
      </script>
  
    </body>
  </html>
  ```

## vue实例 - 元素 - 组件：之间的通信

**v-on，v-bind 是 vue 实例数据和组件之间通信的桥梁**

* **vue 实例**与**元素**绑定，为元素提供**数据**以及**方法**
* 元素可以**获取绑定**的 vue 实例内的**所有东西**
* **元素**想要将东西**传给 vue 实例**，就需要进行 v-bind 之类的操作
* **vue 实例**绑定的元素中的**组件实例**也可以获取 vue 实例的所有东西，**但是**组件（组件只是一个模板）不行
* **组件实例**要想将东西传给**组件**（模板），就需要通过**属性传递**（v-bind，props）和**事件传递**（v-on，this.$emit）一个实例传递，一个组件捕获
* 组件中不能直接用外部（`vue` 实例）的任何元素和方法，只能通过 `v-bind` 和 `v-on` 获取

# 网络通信：Axios

* 从浏览器中创建 `XMLHttpRequests`
* 从 `node.js` 创建 `http` 请求
* 支持 `Promise API` [JS中链式编程]
* 拦截请求和响应
* 转换请求数据和响应数据
* 取消请求
* 自动转换 `JSON` 数据
* 客户端支持防御 `XSRF`（跨站请求伪造）

## 发送请求

* axios

## 获取返回参数

* data() 方法，会自己解析好

## 绑定数据的闪烁问题

### 手动解决

* 没加载之前白屏，使用 `v-clock` 属性

  ```html
  <style>
      [v-clock]{
          display: none;
      }
  </style>
  
  <div v-clock></div>
  ```

* 内容不进行显示，在未加载 vue 数据的时候

# 计算属性

**好处：放在内存中执行，相当于缓存**

* 在实例化中的 `computed` 属性中增加计算 `func`
* 和 `methods` 属性的区别：
  * 在控制台中，`computed` 是一个**属性**，而不是一个方法，不能被调用
  * `computed` 计算出来的**结果**被**缓存**在**内存**中
* `computed` 属性中有内容**修改**（增删改），**缓存失效**，**重新计算**
* 将**不经常变换**的属性放到计算属性中，提高性能

# vue 实例

## vue 实例的基本功能

* 绑定元素，用来提供数据与方法等
  * 数据：传给元素
  * 方法：操作虚拟 dom 等功能，都可以实现
* 发送请求（axios）
* 解析请求（data() 方法）

# vue 项目

## 其他文件引用方法

* 暴漏方法：exports（提供给外部访问，相当于 golang 中的大写，只有大写，外部包才能使用）
* 引用方法：import（导包，整个 js 文件作为一个包，通过**包名**访问其中**暴漏**出来的方法等）

## webpack 打包

* modules 文件存放 js
  * 多个 js 之间互相依赖
  * main.js 为最终的 js，从这个 js 为入口，可以找到其他所有的 js
* webpack.config.js：`webpack` 打包的配置文件
  * js 入口（entry 参数）
  * 打包的输出 `./js/bundle.js`（output 中的 filename，用来生成打包好的 js）

```js
// 将模块导出打包

module.exports = {
    // entry： 入口
    entry: "./modules/main.js",
    output: {
        filename: "./js/bundle.js"
    }
};

// 两种打包方式
// --mode development  开发打包，代码不紧凑，易查看
// --mode production   生产环境打包，压缩代码，体积更小
// 默认 production
```

## vue-router

**路由管理**

