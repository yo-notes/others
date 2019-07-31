# redux notes

[![](https://img.shields.io/badge/redux-v4.0.4-764abc.svg)](https://github.com/reduxjs/redux/releases/tag/v4.0.4)
[![](https://img.shields.io/badge/redux--thunk-v2.3.0-brightgreen.svg)](https://github.com/reduxjs/redux-thunk/releases/tag/v2.3.0)

从一个使用例子看起：

```js
import { createStore } from 'redux'

function counter(state = 0, action) {
  switch (action.type) {
    case 'INCREMENT':
      return state + 1
    case 'DECREMENT':
      return state - 1
    default:
      return state
  }
}

let store = createStore(counter)

store.subscribe(() => console.log(store.getState()))

store.dispatch({ type: 'INCREMENT' }) // 1
store.dispatch({ type: 'INCREMENT' }) // 2
store.dispatch({ type: 'DECREMENT' }) // 1
```

## redux 源码笔记

### createStore(reducer, preloadedState, enhancer)

```js
// 去除了边界等特殊情况处理
export default function createStore(reducer, preloadedState, enhancer) {
  let currentReducer = reducer
  let currentState = preloadedState
  let currentListeners = []
  let nextListeners = currentListeners

  // 获取 state
  function getState() {
    return currentState
  }

  // 注册监听器
  function subscribe(listener) {
    nextListeners.push(listener)

    return function unsubscribe() {
      const index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1)
    }
  }

  // dispatch 会触发所有监听器响应
  function dispatch(action) {
    currentState = currentReducer(currentState, action) // reducer 是一个函数 (state, action) => newState

    const listeners = (currentListeners = nextListeners)
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }
    return action
  }

  dispatch({ type: ActionTypes.INIT })
  return {
    dispatch,
    subscribe,
    getState,
    replaceReducer,
    [$$observable]: observable
  }
}
```

### 中间件

中间件 `redux-thunk` 使用例子：

```js
import { createStore, applyMiddleware } from 'redux';
import thunk from 'redux-thunk';
import rootReducer from './reducers/index';

// Note: this API requires redux@>=3.1.0
const store = createStore(
  rootReducer,
  applyMiddleware(thunk)
);
```

#### 中间件 API 原理

理解中间件之前，先知晓 2 个函数式编程中的概念：柯里化、函数组合，在 `applyMiddleware` 中调用 `compose(...chain)` 会用到。

```js
// compose.js
export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}

// compose(x,y,z)(...args)
// 1. a = x,b = y, return (...args) => x(y(...args))
// 2. a = (...args)=>x(y(...args)), b = z, return (...args)=>x(y(z(...args)))
```

```js
// 返回 createStore => (...args) => {...store, dispatch}
export default function applyMiddleware(...middlewares) {
  return createStore => (...args) => {
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        'Dispatching while constructing your middleware is not allowed. ' +
          'Other middleware would not be applied to this dispatch.'
      )
    }

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```


#### redux-thunk 源码解析


在 createStore 中有这样几行代码（为中间件使用）：

```js
// function createStore(reducer, preloadedState, enhancer)
  if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
    enhancer = preloadedState
    preloadedState = undefined
  }

  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }
    // 这个 enhancer 就是 applyMiddleware(thunk) 返回
    return enhancer(createStore)(reducer, preloadedState)
  }
```

回想中间件使用：

```js
const store = createStore(
  rootReducer,
  applyMiddleware(thunk)
);
```

```js
// redux-thunk 核心
({ dispatch, getState }) => next => action => {
  if (typeof action === 'function') { // 识别中间件
    return action(dispatch, getState, extraArgument);
  }

  return next(action); // next 其实就是 dispatch
};
```

`enhancer(createStore)(reducer, preloadedState)` 相当于 `applyMiddleware(thunk)(createStore)(reducer, preloadedState)`，其中：

* `applyMiddleware(thunk)` 返回 `createStore => (...args) => {...store, dispatch}`
* `applyMiddleware(thunk)(createStore)` 返回 `(...args) => {...store, dispatch}`
* `applyMiddleware(thunk)(createStore)(reducer, preloadedState)`：
  - `const store = createStore(...args)` 以没有 middleware 的情况创建 store 
  - `const chain = middlewares.map(middleware => middleware(middlewareAPI))` 相当于 `chain = [next => action => {}]`
  - `dispatch = compose(...chain)(store.dispatch)` 相当于 `dispatch = (next => action => {})(store.dispatch)`，即 `dispatch = action => {}`，保持 dispatch 的调用格式 `()=>{}`
