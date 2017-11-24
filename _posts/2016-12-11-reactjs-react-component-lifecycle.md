---
title: ReactJS Component 的生命周期
layout: post
category: javascript react.js
tags: react.js react-native
comments: true
---

本文主要是「[8、手把手教React Native实战之ReactJS组件生命周期](https://www.youtube.com/watch?v=3Y5s_WGYi1Y) 」的笔记。

![ReactJS Component 的生命周期 1]({{ site.url }}/assets/post/reactjs-react-component-lifecycle/ReactComponentLifecycle1.jpg)
![ReactJS Component 的生命周期 2]({{ site.url }}/assets/post/reactjs-react-component-lifecycle/ReactComponentLifecycle2.jpg)

## 1. 创建阶段

getDefaultProps：处理 props 的默认值，在 React.createClass 调用

## 2. 实例化阶段

React.render(<HelloMessage />) 启动之后

getInitialState, conpomentWillMount, render, componentDidMount

state：组件的属性，主要是用来存储组件自身需要的数据，每次数据的更新都是通过修改 state 属性的值，ReactJS 内部会间听 state 属性的变化，一旦发生变化的话，就会主动触发组件的 render 方法来更新虚拟 DOM 结构。

虚拟 DOM：将真实的 DOM 结构映射成一个 JSON 数据结构

## 3. 更新阶段

主要发生在用户操作之后或父组件有更新的时候，此时会根据用户的操作行为进行相应的页面结构的调整。

componetWillReceiveProps, shouldComponentUpdate, componentWillUpdate, render, componentDidUpdate

## 4. 销毁阶段

销毁时期被调用，通常做一些取消事件绑定、移除虚拟 DOM 中对应的组件数据结构、销毁一些无效的定时器等工作

``` javascript
var HelloMessage = React.createClass(
  {
  // 1.  创建阶段
    getDefaultProps: function() {
      // 在创建类的时候被调用。this.props 该组件的默认属性
      // （this.props 不能直接修改，只能透过父组件来修改）
      console.log("getDefaultProps");
      return {};
    },

  // 2. 实例化阶段
    getInitialState: function() {
      // 初始化组件的 stete 值，其返回值会赋值给组件的 this.state 属性
      // 获取 this.stte 的默认值
      console.log("getInitialState");
      return {};
    },

    conpomentWillMount: function() {
      // 在 render 之前调用此方法。
      //  业务逻辑的处理都应该放在这里，如对 state 的操作等
      console.log("componentWillMount");
    },

    render: function() {
      // 根据 state 值，渲染并返回一个虚拟 DOM
      console.log("render");
      return <h1 style={{color:'#ff0000', fontSize:'24px'}}>Hello, {this.props.name}！我是东方耀</h1>;
      // 这是注解 React.createElement
    },

    componentDidMount: function() {
      // 该方法发生在 render 方法之后
      // 在该方法中，ReactJS 会使用 render 方法返回的虚拟 DOM 对象来创
      // 建真实的 DOM 结构
      // 组件内部可以通过 this.getDOMNode() 来获取当前组件的节点
      console.log("componentDidMount");
    },

  // 3. 更新阶段，主要发生在用户操作之后或父组件有更新的时候，此时会根据用户的操作行为进行相应的页面结构的调整

    componetWillReceiveProps: function() {
      // 该方法发生在 this.props 被修改成父组件调用 setProps() 方法之后
      // 调用 this.setState 方法来完成对 state 的修改
      console.log("componetWillReceiveProps");
    },

    shouldComponentUpdate: function() {
      // 用来拦截新的 props 或 state，根据逻辑来判断是否需要更新
      console.log("shouldComponentUpdate");

      return true;
    },

    componentWillUpdate: function() {
      // shouldComponentUpdate 返回 true 的时候执行
      // 组件将更新
      console.log("componentWillUpdate");
    },

    componentDidUpdate: function() {
      // 组件更新完毕，我们常在这里做一些 DOM 操作
      console.log("componentDidUpdate");
    },

  // 4. 销毁阶段
    componentWillUmount: function() {
      // 销毁时期被调用，通常做一些取消事件绑定、移除虚拟 DOM 中对应的组件数据结构、销毁一些无效的定时器等工作
      console.log("componentWillUmount");
    }

  }
);

ReactDOM.render(
  <HelloMessage name="React 语法基础 8" />,
  document.getElementById("example")
);
```
