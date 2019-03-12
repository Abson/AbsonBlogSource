---
title: Redux 学习笔记
date: 2018-12-05 11:36:23
tags:
---

`Action` 就是一个描述“发生了什么”的普通对象。比如：
```javascript
 { type: 'LIKE_ARTICLE', articleId: 42 }
```


reducer 是一个纯函数，接收旧的 state 和 action，返回新的 state。
```javascript
(previousState, action) => newState
```
永远不要在 reducer 里做这些操作：

* 修改传入参数；
* 执行有副作用的操作，如 API 请求和路由跳转；
* 调用非纯函数，如 Date.now() 或 Math.random()。

也就是说，请求是放在 `Action` 里面进行的，当 `Action` 内部进行玩异步请求完成后，可通过 `dispatch` 发送完成请求的 `Action`，让 `reducer` 收到，而 reducer 的作用就是返回新的带上了请求返回数据的 `state`，从而达到通知绑定该`state` 的组件

`combineReducers()` 所做的只是生成一个函数，这个函数来调用你的一系列 reducer，每个 reducer 根据它们的 key 来筛选出 state 中的一部分数据并处理，然后这个生成的函数再将所有 reducer 的结果合并成一个大的对象。


Redux 应用中数据的生命周期遵循下面 4 个步骤：
	1. 调用 `store.dispatch(action)`。
	2. Redux store 调用传入的 `reducer` 函数。`Store` 会把两个参数传入 `reducer`： 当前的 state 树和 `action`。
	3. 根 reducer 应该把多个子 reducer 输出合并成一个单一的 state 树。Redux 原生提供 `combineReducers()` 辅助函数
	4. Redux store 保存了根 reducer 返回的完整 state 树。 

