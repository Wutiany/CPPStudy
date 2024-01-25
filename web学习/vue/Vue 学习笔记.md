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

  