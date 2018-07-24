---
title: redux源码阅读
date: 2017-09-19 11:51:23
tags: redux
author: Henry
---
# 前言
Redux是React的一个状态管理工具，或许大家已经比较熟悉了它的使用方式，比如创建store，发起action，使用reducer更新state等等，那么它的内部是如何实现的呢？为了更好的熟悉和使用Redux，我们选择了`3.7.2`版本进行源码阅读，并在这里跟大家分享一下心得。

## 目录结构
redux的代码比较精炼, 按功能分为四部分
* createStore.js 创建store实例
* combineReducers.js 组合多个reducer函数，生成新的reducer
* compose.js & applyMiddleware.js 添加中间件
* bindActionCreators.js 给actions绑定dispatch方法

其中前三个是核心功能，下面我们依次解读源码。

## createStore.js
### createStore
```
export default function createStore(reducer, preloadedState, enhancer) {
  if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
    enhancer = preloadedState
    preloadedState = undefined
  }

  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }

    return enhancer(createStore)(reducer, preloadedState)
  }
```
它接收三个参数:
* reducer **`[Function]`** 顶层reducer, 接收store的全局state和待处理的action, 返回新的全局state
* preloadedState **`[Object]`** 初始化的state对象
* enhancer **`[Function]`** 添加中间件, 由applyMiddleware生成

在createStore函数内, redux定义了一系列的变量和方法, 用于缓存数据和触发事件  

### getState & subscribe

* getState(): 获取全局state
* subscribe(): 注册监听函数


```
  let currentReducer = reducer
  let currentState = preloadedState
  let currentListeners = []
  let nextListeners = currentListeners
  let isDispatching = false
	
  function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice()
    }
  }
	
  /**
   * Reads the state tree managed by the store.
   *
   * @returns {any} The current state tree of your application.
   */
  function getState() {
    return currentState
  }
	
  /**
   * Adds a change listener. It will be called any time an action is dispatched,
   * and some part of the state tree may potentially have changed. You may then
   * call `getState()` to read the current state tree inside the callback.
   *
   ......
	 
   *
   * @param {Function} listener A callback to be invoked on every dispatch.
   * @returns {Function} A function to remove this change listener.
   */
  function subscribe(listener) {
    if (typeof listener !== 'function') {
      throw new Error('Expected listener to be a function.')
    }

    let isSubscribed = true

    ensureCanMutateNextListeners()
    nextListeners.push(listener)

    return function unsubscribe() {
      if (!isSubscribed) {
        return
      }

      isSubscribed = false

      ensureCanMutateNextListeners()
      const index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1)
    }
  }
```

这里缓存了三个数据: 
* currentState: 全局state
* currentListeners: 已生效的监听函数队列
* nextListeners: 即将生效的监听函数队列


每当改变监听函数队列时, 都会调用 ensureCanMutateNextListeners, 以确保对于 nextListeners 的改变不会影响 currentListeners  
调用 store.dispatch 时, 会把 nextListeners 赋值给 currentListeners

### dispatch

这是最原始的dispatch函数 ( 为什么说是最原始的, 看到中间件的时候就会明白了 )  

其功能如下: 
* 检查 action 类型
* 调用顶层 reducer, 获取新的全局state
* 调用监听函数队列


````
  /**
   * Dispatches an action. It is the only way to trigger a state change.
   ......
	 
   * @param {Object} action A plain object representing “what changed”. It is
   * a good idea to keep actions serializable so you can record and replay user
   * sessions, or use the time travelling `redux-devtools`. An action must have
   * a `type` property which may not be `undefined`. It is a good idea to use
   * string constants for action types.
   ......
	 
   * @returns {Object} For convenience, the same action object you dispatched.
   */
  function dispatch(action) {
    if (!isPlainObject(action)) {
      throw new Error(
        'Actions must be plain objects. ' +
        'Use custom middleware for async actions.'
      )
    }

    if (typeof action.type === 'undefined') {
      throw new Error(
        'Actions may not have an undefined "type" property. ' +
        'Have you misspelled a constant?'
      )
    }

    if (isDispatching) {
      throw new Error('Reducers may not dispatch actions.')
    }

    try {
      isDispatching = true
      currentState = currentReducer(currentState, action)
    } finally {
      isDispatching = false
    }

    const listeners = currentListeners = nextListeners
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }

    return action
  }
````

createStore执行完后, 发出初始化的action
```
// When a store is created, an "INIT" action is dispatched so that every
  // reducer returns their initial state. This effectively populates
  // the initial state tree.
  dispatch({ type: ActionTypes.INIT })
```

## combineReducers.js
在执行核心代码前，做了许多校验和错误处理
### combineReducers
接受一个对象，返回一个函数，主要逻辑如下：
* 处理传入的对象, 将所有值为函数的属性，缓存为reducers对象
* 返回一个combine函数，作为新的reducer，接收 state & action
* 当执行combine时，调用内部缓存的reducers对象，以属性名为键值，将所有reducer的执行结果保存为一个对象
* 判断 state 是否改变，决定返回新的 nextState 或者传入的 state

```
/**
 * Turns an object whose values are different reducer functions, into a single
 * reducer function. It will call every child reducer, and gather their results
 * into a single state object, whose keys correspond to the keys of the passed
 * reducer functions.
 */
export default function combineReducers(reducers) {
  const reducerKeys = Object.keys(reducers)
  const finalReducers = {}
	// 获取所有值为函数的属性，组成新 reducers 对象
  for (let i = 0; i < reducerKeys.length; i++) {
    const key = reducerKeys[i]

    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key]
    }
  }
  const finalReducerKeys = Object.keys(finalReducers)

  let unexpectedKeyCache
  if (process.env.NODE_ENV !== 'production') {
    unexpectedKeyCache = {}
  }

  // 返回新的 reducer 函数
  return function combination(state = {}, action) {
    let hasChanged = false
    const nextState = {}
		// 遍历 reducers 对象, 将每个函数的结果保存到 nextState 中
    for (let i = 0; i < finalReducerKeys.length; i++) {
      const key = finalReducerKeys[i]
      const reducer = finalReducers[key]
      const previousStateForKey = state[key]
      const nextStateForKey = reducer(previousStateForKey, action)
      if (typeof nextStateForKey === 'undefined') {
        const errorMessage = getUndefinedStateErrorMessage(key, action)
        throw new Error(errorMessage)
      }
      nextState[key] = nextStateForKey
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
    }
		// 如果 nextState 改变, 返回新的 state
    return hasChanged ? nextState : state
  }
}
```

## bindActionCreators.js
```
function bindActionCreator(actionCreator, dispatch) {
  return (...args) => dispatch(actionCreator(...args))
}

/**
 * Turns an object whose values are action creators, into an object with the
 * same keys, but with every function wrapped into a `dispatch` call so they
 * may be invoked directly. This is just a convenience method, as you can call
 * `store.dispatch(MyActionCreators.doSomething())` yourself just fine.
 */
export default function bindActionCreators(actionCreators, dispatch) {
  if (typeof actionCreators === 'function') {
    return bindActionCreator(actionCreators, dispatch)
  }

  const keys = Object.keys(actionCreators)
  const boundActionCreators = {}
  for (let i = 0; i < keys.length; i++) {
    const key = keys[i]
    const actionCreator = actionCreators[key]
    if (typeof actionCreator === 'function') {
      boundActionCreators[key] = bindActionCreator(actionCreator, dispatch)
    }
  }
  return boundActionCreators
}
```

这个就比较简单了，actionCreator(s) => 绑定dispatch的actionCreator(s)
* 判断 actionCreators 类型，若为函数，直接返回绑定后的 function
* 若 actionCreators 为对象，依次给属性对应的函数绑定dispatch，并返回新生成的map对象
