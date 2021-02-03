
## 前言
中间件的内容其实不属于`React`源码相关，属于`Redux`相关。但是中间件的原理是一个非常重要的知识点，它是我们前端开发解决一些业务问题时的利器。很多前端应用的架构都是使用中间件为基础搭建的。

本文不会介绍`Redux`相关的内容，只关注于中间件的实现原理。

本文将是我学习`react`源码的目前阶段的最后一篇文章，`react`源码内容比较多，也比较晦涩，我也不能够一蹴而就。等过段时间再继续深入学习的时候，再来接着更新这一系列的内容。

## 认识Redux中间件
在`React`开发中，管理数据状态的`Redux`是每个人都会接触到的内容（如同`Vuex`在`Vue`开发当中的地位）。我们知道，在`Redux`中，我们想要修改数据，我们必须先派发一个`Action`，`Action`会被`Reducer`读取，`Reducer`将根据`Action`内容的不同执行不同的计算逻辑，最终生成新的`state`，这个新的`state`会更新到`Store`对象里，进而驱动视图层面作出对应的改变。

这里有一个需要注意的地方，`Redux`源码中只有同步操作，也就是说当我们`dispatch action`时，`state`会被立即更新。

如果我们想引入异步数据流，该怎么办？官方的建议就是使用中间件。

本文使用`redux-thunk`作为处理异步数据流的方式。[源码地址](https://github.com/reduxjs/redux-thunk/blob/master/src/index.js)

```js
import thunkMiddleware from 'redux-thunk'
import reducer from './reducers'
// 使用redux-thunk中间件
const store = createStore(reducer, applyMiddleware(thunkMiddleware))
```

这样配置之后，我们就可以给`dispatch`（这个`dispatch`并非原始的`dispatch`）传入一个函数，并可以在该函数中使用异步数据流。

## thunk中间件
`thunk`的源码很简单，也就10多行代码
```js
function createThunkMiddleware(extraArgument) {
  // 返回值是一个 thunk，它是一个函数
  return ({ dispatch, getState }) => (next) => (action) => {
    // thunk 若感知到 action 是一个函数，就会执行 action
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }
    // 若 action 不是一个函数，则不处理，直接放过
    return next(action);
  };
}
const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;
export default thunk;
```
在这里先简单的分析一下它的源码。首先这个方法返回是一个三重的函数，第一重接收的是一个`dispatch`（组合中间件出来的一个链）和`getState`（`store.getState`获取`state`数据的方法）的对象，第二重是`next`（`store.dispatch`最原始的`dispatch`方法），第三重是`action`（这个方法是一个`action`对象或者是一个异步的函数）。

这里的内容可能一下子无法理解，没关系，接下来我们配合`Redux`的`applyMiddleware`的方法流程，一步步来剖析`Redux`的中间件原理。

## Redux的中间件原理

我们先看看`applyMiddleware`的源码。

```js
// applyMiddlerware 会使用“...”运算符将入参收敛为一个数组
export default function applyMiddleware(...middlewares) {
  // 它返回的是一个接收 createStore 为入参的函数
  return createStore => (...args) => {
    // 首先调用 createStore，创建一个 store
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        `Dispatching while constructing your middleware is not allowed. ` +
          `Other middleware would not be applied to this dispatch.`
      )
    }

    // middlewareAPI 是中间件的入参
    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    // 遍历中间件数组，调用每个中间件，并且传入 middlewareAPI 作为入参，得到目标函数数组 chain
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    // 改写原有的 dispatch：将 chain 中的函数按照顺序“组合”起来，调用最终组合出来的函数，传入 dispatch 作为入参
    dispatch = compose(...chain)(store.dispatch)

    // 返回一个新的 store 对象，这个 store 对象的 dispatch 已经被改写过了
    return {
      ...store,
      dispatch
    }
  }
}
```
因为`applyMiddleware`是在`createStore`当中使用的，所以我们也需要看一部分的`createStore`源码。
```js
function createStore(reducer, preloadedState, enhancer) {
    // 这里处理的是没有设定初始状态的情况，也就是第一个参数和第二个参数都传 function 的情况
    if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
        // 此时第二个参数会被认为是 enhancer（中间件）
        enhancer = preloadedState;
        preloadedState = undefined;
    }
    // 当 enhancer 不为空时，便会将原来的 createStore 作为参数传入到 enhancer 中
    if (typeof enhancer !== 'undefined') {
        return enhancer(createStore)(reducer, preloadedState);
    }
    ......
}
```

`createStore`对于`applyMiddleware`的使用逻辑比较简单其实主要就是一句话
```js
return enhancer(createStore)(reducer, preloadedState);
```

我们把这几个参数，代入`applyMiddleware`中再去看。
```js
// middlewares是传入的中间件
export default function applyMiddleware(...middlewares) {
  // 它返回的是一个接收 createStore 为入参的函数
  return createStore => (...args) => {
    ......
  }
}
```
`applyMiddleware`方法中的`createStore`，其实就是`Redux`中的`createStore`方法，而`args`则对应的是`reducer`、`preloadedState`，这两个参数均为`createStore`函数的约定入参。

我们接着来看下面的内容
```js
// 首先调用 createStore，创建一个 store
const store = createStore(...args)
// 用来防止在遍历 middleware 时调用dispatch
let dispatch = () => {
  throw new Error(
    `Dispatching while constructing your middleware is not allowed. ` +
      `Other middleware would not be applied to this dispatch.`
  )
}

// middlewareAPI 是中间件的入参
const middlewareAPI = {
  getState: store.getState,
  dispatch: (...args) => dispatch(...args)
}
// 遍历中间件数组，调用每个中间件，并且传入 middlewareAPI 作为入参，得到目标函数数组 chain
const chain = middlewares.map(middleware => middleware(middlewareAPI))
```

这里的内容就是调用`createStore`创建一个`store`。然后创建一个`middlewareAPI`对象，遍历`middleware`（中间件），并传入`middlewareAPI`参数。这里其实就是之前`thunk`第一重函数接收的`dispatch`和`getState`。

然后看下一句
```js
dispatch = compose(...chain)(store.dispatch)
```

这里的`compose(...chain)`我们下面的章节再看，这里可以先把这个方法看成如下
```js
dispatch = middleware(middlewareAPI)(store.dispatch)
```

这里正好对于`thunk`的第二重函数接收`next`参数。**这里要注意一点，`dispatch`已经被重写。**

```js
return {
  ...store,
  dispatch
}
```
最后返回一个重写过`dispatch`的对象，这个对象会被绑定到我们的页面上，比如
```js
<Provider store={store}>
  <App />
</Provider>
```
然后我们通过`useDispatch`拿到的`dispatch`其实就是`thunk`中间件改写过的方法。我们用这个`dispatch`去传入`action`对象或者异步数据流函数，其实就是调用`thunk`的第三重函数
```js
(action) => {
  // thunk 若感知到 action 是一个函数，就会执行 action
  if (typeof action === 'function') {
    return action(dispatch, getState, extraArgument);
  }
  // 若 action 不是一个函数，则直接使用dispatch派发action
  return next(action);
};
```
注意一点，这里的`dispatch`已经是被改写的`dispatch = compose(...chain)(store.dispatch)`。

这样，我们就通过中间件实现了`Redux`的异步数据流。我们就可以给`dispatch`传入一个函数，来异步的派发`action`。

## compose
`compose`是一个函数式编程中，很常用的工具方法。它的功能就是把多个函数组合起来。

在`redux`中若有多个中间件，那么`redux`会结合它们被“安装”的先后顺序，依序调用这些中间件。所以，我们需要使用`compose`方法把中间件函数组合起来。

**注意:`compose`只是一个函数式编程的思路，实现方式有很多种，下面只是`redux`中实现的一种**
```js
port default function compose(...funcs) {
  // 处理数组为空的边界情况
  if (funcs.length === 0) {
    return arg => arg
  }

  // 若只有一个函数，也就谈不上组合，直接返回
  if (funcs.length === 1) {
    return funcs[0]
  }
  // 若有多个函数，那么调用 reduce 方法来实现函数的组合
  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}

compose(f1, f2, f3, f4);
// => 会转换成下面这种形式
(...args) =>  f1(f2(f3(f4(...args))));
```

多个中间件嵌套的时候，这里的思路可能会有点绕，我举个例子来说明一下。假设我们使用了两个中间件，一个是`thunk`，一个是`redux-logger`（当`redux`数据变更的时候，会打印相关信息到控制台）。[源码地址](https://github.com/LogRocket/redux-logger/blob/master/src/index.js)
```js
// dispatch定义
dispatch = thunk(createLogger(store.dispatch));
// dispatch被使用
dispatch(action);
//等同于
thunk(createLogger(store.dispatch))(action)
```
我们来看下`redux-logger`的源码，如果我只保留与`redux`中间件相关的逻辑的话，它的源码可以压缩成几行代码。
```js
return ({ getState }) => next => (action) => {
  return next(action);
};
```

这里也有三重的代码，第一重在我们遍历中间件数组的时候就被调用了
```js
const chain = middlewares.map(middleware => middleware(middlewareAPI));
```

所以这里传入`thunk`的应该是第二重的代码
```js
thunk(createLogger(store.dispatch))(action)
// 等同于
thunk((action) => store.dispatch(action))(action)
```
我们再来看看之前`thunk`中间件的源码
```js
function createThunkMiddleware(extraArgument) {
  // 返回值是一个 thunk，它是一个函数
  return ({ dispatch, getState }) => (next) => (action) => {
    // thunk 若感知到 action 是一个函数，就会执行 action
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }
    // 若 action 不是一个函数，则不处理，直接放过
    return next(action);
  };
}
const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;
export default thunk;
```

我们再次来精简代码
```js
thunk((action) => store.dispatch(action))(action)
// 等同于
const thunk = (next) => (action) => {
  if (typeof action === 'function') {
    return action(dispatch, getState, extraArgument);
  }
  return next(action);
};
thunk((action) => store.dispatch(action))(action);
// 等同于
const thunk = (action) => {
  if (typeof action === 'function') {
    return action(dispatch, getState, extraArgument);
  }
  return store.dispatch(action);
};
thunk(action);
```
这样，就实现了多个中间件的依次调用。

## 感谢
如果本文对你有所帮助，请帮忙点个赞，感谢！