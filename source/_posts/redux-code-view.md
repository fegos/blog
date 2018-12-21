---
title: Redux源码解析
date: 2018-12-05
tags: redux
author: henry
---

Redux是一个状态管理工具，它维护全局的`store`, 通过派发`action`利用`reducer`更新`state`.

## 工作流

我们先通过一段代码, 看一下redux在实际项目中是如何使用的

**实例化Store**
```javascript
import { createStore, applyMiddleware } from 'redux';
import thunk from 'redux-thunk';
import reducers from './reducer';

// 初始化state
const initialState = { ... };

// 自定义中间件
const logger = store => next => action => {
  console.info('触发行为', action);
  return next(action);
};

// 实例化store
export default createStore(
  reducers,
  initialState,
  applyMiddleware( thunk, logger ),
);
```

**入口模块**, 通过Provider绑定全局store

```javascript
/* index.js */
import { Provider } from 'react-redux';
import store from './store';

class App extends Component {
  render() {
    return (
      <Provider store={store}>
        <Router history={history} routes={routes} />
      </Provider>
    )
  }
}
```

**业务模块**, 使用connect注册当前页面, 获取state和dispatch

```javascript
/* page */
import React, { Component } from 'react'
import { connect } from 'react-redux'
import { bindActionCreators } from 'redux'
import actions from './action'

class Page extends Component {

  /* ... */

  componentDidMount() {
    const { page, actions } = this.props;
  }
}

// 注册模块, 获得state & dispatch
export default connect(
  state => ({ page: state.page }),
  dispatch => ({ actions: bindActionCreators(actions, dispatch) })
)(Page)
```

通过一张图了解Redux是如何工作的

![image](/images/redux-react.png)

**reducer.js**
```javascript
import { combineReducers } from 'redux'
import { USER_MANAGER_ADD_XXX } from './action'

function reducerFn(state={}, action) {
  switch (action.type) {
    case USER_MANAGER_ADD_XXX:
      return {
        ...state,
        ...action.payload
      }
    default:
      return state
  }
}

// 通过combineReducers合并多个reducer, 返回包装后的reducer
export default combineReducers({
   reducerFn,
   /* more reducers */
})
```

**action.js**
```javascript
export const USER_MANAGER_ADD_XXX = "USER_MANAGER_ADD_XXX"

// action creator
function add_XXX(param) { 
  // 返回一个action
  return {
    type: USER_MANAGER_ADD_XXX,
    payload: {}
  }
}

export default {
  add_XXX
}
```

## Redux源码

redux的代码比较精炼, 按功能分为四部分
* createStore.js 创建`store`, 注册监听函数, 提供原始`dispatch`函数等
* combineReducers.js `reducer`的包装函数, 更新`state`
* applyMiddleware.js `store`的修饰函数, 注册中间件
* compose.js 组合函数, 接受函数数组, 返回一个包装后的函数
* bindActionCreators.js 给`actionCreator`绑定`dispatch`

其中前三个是核心功能，下面我们依次解读源码。

### createStore.js

提供`createStore`方法, 创建新的`store`对象

```javascript
export default function createStore(reducer, preloadedState, enhancer) {
  // 处理参数顺序
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

  /* ... */

  return {
    dispatch,
    subscribe,
    getState,
    ...
  }
}
```

它接收三个参数:
* `reducer`, 包装后的`reducer`函数
* `preloadedState`, 初始化的`state`对象
* `enhancer`, `store`的修饰函数, 添加中间件

在`createStore`中定义了一些重要的方法和属性

**getState**

获取当前`store`维护的数据对象`state`

```javascript
/**
  * Reads the state tree managed by the store.
  *
  * @returns {any} The current state tree of your application.
  */
function getState() {
  return currentState
}
```

**dispatch**

原始dispatch函数, 其功能如下: 
* 检查`action`类型
* 调用包装后的`reducer`, 更新`state`
* 调用监听函数队列

```javascript
let currentReducer = reducer    // 包装后的reducer入口函数
let currentState = preloadedState   // 当前store维护的数据对象
let currentListeners = []   // 当前的监听函数队列
let nextListeners = currentListeners    // 即将生效的监听函数队列
let isDispatching = false   // 派发事件的状态

function dispatch(action) {
  // 处理action, 更新state
  try {
    isDispatching = true
    currentState = currentReducer(currentState, action)
  } finally {
    isDispatching = false
  }
  // 调用所有的监听函数, 触发回调事件
  const listeners = (currentListeners = nextListeners)
  for (let i = 0; i < listeners.length; i++) {
    const listener = listeners[i]
    listener()
  }

  return action
}
```

**subscribe**

注册的回调函数, 监听`dispatch`事件

```javascript
function ensureCanMutateNextListeners() {
  if (nextListeners === currentListeners) {
    nextListeners = currentListeners.slice()
  }
}

function subscribe(listener) {
  // 私有的注册状态
  let isSubscribed = true
  // 确保即将生效的监听函数队列可用
  ensureCanMutateNextListeners()
  nextListeners.push(listener)

  // 返回监听事件的注销函数
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

每当注册监听函数时, 会将其加入`nextListeners`队列; 在下次调用`dispatch`时, 将`nextListeners`的值更新到当前的监听函数队列`currentListeners`, 确保了每次`dispatch`不会影响到之后注册的`listener`.

`createStore`调用完成之前, 会发出一个初始化`action`, 用于生成初始化的`state`.
```javascript
// When a store is created, an "INIT" action is dispatched so that every
// reducer returns their initial state. This effectively populates
// the initial state tree.
dispatch({ type: ActionTypes.INIT })
```

### combineReducers.js

该模块的核心功能如下:
* 将多个`reducers`包装成一个新的`combine`函数, 接受两个参数`state`和`action`
* 以`reducers`的函数名为`key`, 返回值为值, 生成对应的`state`对象
* 被调用时, 将`state`根据`key`, 拆分成多个子`state`, 与`action`一起传递给对应的子`reducer`函数
* `combine`函数可以作为`reducer`, 通过`combineReducers`生成新的`combine`, 形成`state tree`

```javascript
export default function combineReducers(reducers) {
  const reducerKeys = Object.keys(reducers)
  const finalReducers = {} // 当前层级的reducer集合

  for (let i = 0; i < reducerKeys.length; i++) {
    const key = reducerKeys[i]
    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key]
    }
  }

  const finalReducerKeys = Object.keys(finalReducers)

  // 返回新的reducer函数
  return function combination(state = {}, action) {
    let hasChanged = false
    const nextState = {}

    // 处理所有的reducer, 更新当前层级的state对象
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
    // hasChanged为true, 则返回新的state
    return hasChanged ? nextState : state
  }
}
```

### applyMiddleware.js

通过修饰函数, 为`store`添加中间件, 并在`dispatch`调用时依次执行

```javascript
export default function applyMiddleware(...middlewares) {
  // 返回store的修饰函数
  return createStore => (...args) => {
    const store = createStore(...args)

    // 初始化final dispatch
    let dispatch = () => {
      throw new Error(
        `Dispatching while constructing your middleware is not allowed. ` +
          `Other middleware would not be applied to this dispatch.`
      )
    }

    // 中间件参数
    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args) // 封装一层的final dispatch, 作为中间件中的dispatch函数
    }

    // 初始化的中间件队列
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    // 中间件变为逐层调用的组合关系, 并包装原始的dispatch, 赋值给final dispatch
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}

/* import from compose.js */
function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }
  // 数组变为组合: [f, b, c, ...] => f(b(c(...)))
  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```

* `store`上的原始`dispatch`函数, 经过中间件的包装, 生成了新的`dispatch`函数
* `applyMiddleware`返回的修饰函数, 将`store.dispatch`替换成包装后的`dispatch`, 并返回新的`store`
* 中间件的调用顺序为, 从左到右依次调用

![image](/images/redux-middleware.png)

**Middleware**

中间件的结构较为复杂, 它是由多层函数的**柯里化**构成, 我们以`redux-thunk`为例了解其实现机制

```javascript
/* redux-thunk */
function createThunkMiddleware(extraArgument) {
  // 返回中间件
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
  };
}

const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;

export default thunk;
```

* 包装层, 获得`store`实例, 在applyMiddleware中被调用, 生成用于中间件链式调用的传递函数
* 传递层, 获得钩子函数`next`, 返回自身的调用钩子, 作为参数传递给下个中间件的传递函数
* 逻辑层, 钩子函数（包装后的`dispatch`）, 获得`action`, 处理中间件业务逻辑, 调用上一个中间件的钩子`next(action)`, 形成链式调用直至`store.dispatch(action)`

### bindActionCreators.js

给`actionCreator`绑定`dispatch`函数

```javascript
function bindActionCreator(actionCreator, dispatch) {
  // 返回绑定后的函数, 可以直接调用, 派发action
  return (...args) => dispatch(actionCreator(...args))
}

export default function bindActionCreators(actionCreators, dispatch) {
  if (typeof actionCreators === 'function') {
    return bindActionCreator(actionCreators, dispatch)
  }

  const keys = Object.keys(actionCreators)
  const boundActionCreators = {}
  // 逐个绑定actionCreator
  for (let i = 0; i < keys.length; i++) {
    const key = keys[i]
    const actionCreator = actionCreators[key]
    if (typeof actionCreator === 'function') {
      boundActionCreators[key] = bindActionCreator(actionCreator, dispatch)
    }
  }
  // 返回绑定后的actionCreators对象, 结构与之前一样
  return boundActionCreators
}
```

这个概念就比较简单了
* 判断`actionCreators`类型，若为函数，直接返回绑定后的`function`
* 若`actionCreators`为对象，依次给属性对应的函数绑定`dispatch`，并返回新生成的map对象
* 绑定后的`actionCreator`, 可以直接调用并派发`action`

## react-redux

`Redux`是一个独立的数据管理工具, 如果想让它和`React`一起工作, 需要引入`react-redux`模块, 它主要提供`Provider`和`connect`两个API.

**Provider**

本身为`ReactComponent`, 通过`context`为子元素提供`store`对象

```javascript
export default class Provider extends Component {
  getChildContext() {
    return { store: this.store }
  }

  constructor(props, context) {
    super(props, context)
    this.store = props.store
  }

  render() {
    // 只允许有一个子元素
    return Children.only(this.props.children)
  }
}

Provider.propTypes = {
  store: storeShape.isRequired,
  children: PropTypes.element.isRequired
}

Provider.childContextTypes = {
  store: storeShape.isRequired
}
```

**connect**

修饰函数, 为`ReactComponent`提供`state`和`dispatch`, 并将修饰后的属性合并到`ReactComponent`的`props`上

```javascript
export default function connect(mapStateToProps, mapDispatchToProps, mergeProps, options = {}) {

  /* ... */

  // 包装函数
  return function wrapWithConnect(WrappedComponent) {
    return class Connect extends Component {
      constructor(props, context) {
        super(props, context)
        this.version = version
        this.store = props.store || context.store   // 获取store

        const storeState = this.store.getState()
        this.state = { storeState }   // 初始化state
        this.clearCache()
      }
      computeStateProps() { ... }
      computeDispatchProps() { ... }
      render() {
        /* ... */
        this.renderedElement = createElement(WrappedComponent,
          this.mergedProps
        );
        return this.renderedElement
      }
      trySubscribe() {
        if (shouldSubscribe && !this.unsubscribe) {
          this.unsubscribe = this.store.subscribe(this.handleChange.bind(this))
          this.handleChange()
        }
      }
      tryUnsubscribe() {
        if (this.unsubscribe) {
          this.unsubscribe()
          this.unsubscribe = null
        }
      }
      handleChange() {
        if (!this.unsubscribe) return
        const storeState = this.store.getState()
        const prevStoreState = this.state.storeState
        if (prevStoreState === storeState) return
        this.hasStoreStateChanged = true
        this.setState({ storeState })
      }
    }
  }
}
```

**react + redux**

在入口文件中使用`Provider`, 为`App`中的所有子元素提供`store`对象
```javascript
import React from "react";
import ReactDOM from "react-dom";

import { Provider } from "react-redux";
import store from "./store";

import App from "./App";

const rootElement = document.getElementById("root");
ReactDOM.render(
  <Provider store={store}>
    <App />
  </Provider>,
  rootElement
);
```

对需要用到`redux`状态管理的`ReactComponent`使用`connect`进行包装, 通过`mapStateToProps`和`mapDispatchToProps`参数/方法, 将处理后的`state`和`dispatch`合并到`ReactComponent`的`props`上

```javascript
import { connect } from "react-redux";
import { increment, decrement, reset } from "./actionCreators";

// const Counter = ...

const mapStateToProps = (state /*, ownProps*/) => {
  return {
    counter: state.counter
  };
};

/*
 * 该参数为对象时, react-redux内部通过bindActionCreators绑定dispatch
 * 该参数为函数时, 接受dispatch作为参数, 返回已绑定dispatch的actions
**/
const mapDispatchToProps = { increment, decrement, reset };

export default connect(
  mapStateToProps,
  mapDispatchToProps
)(Counter);
```
