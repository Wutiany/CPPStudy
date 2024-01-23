# Ajax 基础学习笔记

## 三部曲

1. 编写对应处理 Controller，返回消息或者字符串或者json格式的数据
2. 编写 ajax 请求
   * url：controller 请求
   * data：键值对
   * success：回调函数
3. 给 ajax 绑定时间，点击 `.click`, 失去焦点 `onblur`，键盘弹起 `keyup`

## 后端理解

首先 ajax 相当于向后端发送请求，然后在当前页面显示

ajax 功能：

1. 向指定 url 发送请求（$.ajax 默认是 GET 请求）
2. 发送指定的请求数据
3. 回调函数，获取服务端返回的数据，进行操作（处理、显示给前端，操作前端的组件，可以使用一些 js 的功能进行处理）

