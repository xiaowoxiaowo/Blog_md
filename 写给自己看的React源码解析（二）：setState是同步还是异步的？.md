## 前言
这个问题相信是大部分的人刚开始学习`React`的时候，第一个碰到的问题，对我来说，学习的教程里就只是告诉我直接使用`setState`是异步的，但是在一些比如`setTimeout`这样的异步方法里，它是同步的。

我那时候就很疑惑，虽然想要一探究竟，但是为了尽快上手`React`，还是以应用优先，留到之后的源码学习时，再来深入了解一下。

本文的内容就是从源码层面来分析`setState`到底是同步还是异步的。因为现在做的项目其实都是使用`hooks`在维护数据状态，对于`class`使用特别少，所以我并不会太过深究底层的渲染原理，重点只是在于为什么`setState`有的时候表现是同步，有的时候表现是异步。

## 例子
```js
export default class App extends React.Component{
  state = {
    num: 0
  }
  add = () => {
    console.log('add前', this.state.num)
    this.setState({
      num: this.state.num + 1
    });
    console.log('add后', this.state.num)
  }
  add3 = () => {
    console.log('add3前', this.state.num)
    this.setState({
      num: this.state.num + 1
    });
    this.setState({
      num: this.state.num + 1
    });
    this.setState({
      num: this.state.num + 1
    });
    console.log('add3后', this.state.num)
  }
  reduce = () => {
    setTimeout(() => {
      console.log('reduce前', this.state.num)
      this.setState({
        num: this.state.num - 1
      });
      console.log('reduce后', this.state.num)
    },0);
  }
  render () {
    return <div>
      <button onClick={this.add}>点击加1</button>
      <button onClick={this.add3}>点击加3次</button>
      <button onClick={this.reduce}>点我减1</button>
    </div>
  }
}
```
按顺序依次点击这三个按钮，我们来看下控制台打印出来的内容。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d817aa034dd40e5a718dbc2f0ccf7c0~tplv-k3u1fbpfcp-watermark.image)

这个结果，对于有点经验`React`开发者来说很简单。

按照这个例子来看，**`setState`是一个异步的方法，执行完成之后，数据不会马上修改，会等到后续某个时刻才进行变化。多次调用`setState`，只会执行最新的那个事件。在异步的方法中，它会有同步的特性。**

我们先不着急下结论，我们深入`setState`的流程中去找结论。

## 异步的原理-批量更新

出于性能优化的需要，一次`setState`是不会触发一个完整的更新流程的，在一个同步的代码运行中，每次执行一个`setState`,`React`会把它塞进一个队列里，等时机成熟，再把“攒起来”的`state`结果做合并，最后只针对最新的`state`值走一次更新流程。这个过程，叫作**批量更新**。

这样子，就算我们代码写的再烂，比如写了一个循环100次的方法，每次都会调用一个`setState`，也不会导致频繁的`re-render`造成页面的卡顿。

这个原理，解释了上面第一个按钮以及第二个按钮的现象。

## 同步的原理-setState工作流

这里的问题就一个，为什么`setTimeout`可以将`setState`的执行顺序从异步变为同步？

我们来看看`setState`的源码

```js
ReactComponent.prototype.setState = function (partialState, callback) {
  this.updater.enqueueSetState(this, partialState);
  if (callback) {
    this.updater.enqueueCallback(this, callback, 'setState');
  }
};
```
不考虑`callback`回调,这里其实就是触发了一个`enqueueSetState`方法。

```js
enqueueSetState: function (publicInstance, partialState) {
  // 根据 this 拿到对应的组件实例
  var internalInstance = getInternalInstanceReadyForUpdate(publicInstance, 'setState');
  // 这个 queue 对应的就是一个组件实例的 state 数组
  var queue = internalInstance._pendingStateQueue || (internalInstance._pendingStateQueue = []);
  queue.push(partialState);
  //  enqueueUpdate 用来处理当前的组件实例
  enqueueUpdate(internalInstance);
}
```
这个方法就是刚才说的，把`state`的修改放进队列中。然后使用`enqueueUpdate`来处理将要更新的组件实例。再来看看`enqueueUpdate`方法。
```js
function enqueueUpdate(component) {
  ensureInjected();
  // 注意这一句是问题的关键，isBatchingUpdates标识着当前是否处于批量创建/更新组件的阶段
  if (!batchingStrategy.isBatchingUpdates) {
    // 若当前没有处于批量创建/更新组件的阶段，则立即更新组件
    batchingStrategy.batchedUpdates(enqueueUpdate, component);
    return;
  }
  // 否则，先把组件塞入 dirtyComponents 队列里，让它“再等等”
  dirtyComponents.push(component);
  if (component._updateBatchNumber == null) {
    component._updateBatchNumber = updateBatchNumber + 1;
  }
}
```

这里重点关注一个对象，`batchingStrategy`（`React`内部专门用于管控批量更新的对象），它的属性`isBatchingUpdates`直接决定了当下是否要走更新流程，还是应该等等。

每当`React`调用`batchedUpdate`去执行更新动作时，会先把这个锁给“锁上”（置为`true`），表明“现在正处于批量更新过程中”。当锁被“锁上”的时候，任何需要更新的组件都只能暂时进入`dirtyComponents`里排队等候下一次的批量更新。

```js
var ReactDefaultBatchingStrategy = {
  // 全局唯一的锁标识
  isBatchingUpdates: false,
 
  // 发起更新动作的方法
  batchedUpdates: function(callback, a, b, c, d, e) {
    // 缓存锁变量
    var alreadyBatchingStrategy = ReactDefaultBatchingStrategy. isBatchingUpdates
    // 把锁“锁上”
    ReactDefaultBatchingStrategy. isBatchingUpdates = true

    if (alreadyBatchingStrategy) {
      callback(a, b, c, d, e)
    } else {
      // 启动事务，将 callback 放进事务里执行
      transaction.perform(callback, null, a, b, c, d, e)
    }
  }
}
```
这里，我们还需要了解`React`中的`Transaction`（事务） 机制。

`Transaction`在`React`源码中表现为一个核心类，`Transaction`可以创建一个黑盒，该黑盒能够封装任何的方法。因此，那些需要在函数运行前、后运行的方法可以通过此方法封装（即使函数运行中有异常抛出，这些固定的方法仍可运行），实例化`Transaction`时只需提供相关的方法即可。

```js
* <pre>
 *                       wrappers (injected at creation time)
 *                                      +        +
 *                                      |        |
 *                    +-----------------|--------|--------------+
 *                    |                 v        |              |
 *                    |      +---------------+   |              |
 *                    |   +--|    wrapper1   |---|----+         |
 *                    |   |  +---------------+   v    |         |
 *                    |   |          +-------------+  |         |
 *                    |   |     +----|   wrapper2  |--------+   |
 *                    |   |     |    +-------------+  |     |   |
 *                    |   |     |                     |     |   |
 *                    |   v     v                     v     v   | wrapper
 *                    | +---+ +---+   +---------+   +---+ +---+ | invariants
 * perform(anyMethod) | |   | |   |   |         |   |   | |   | | maintained
 * +----------------->|-|---|-|---|-->|anyMethod|---|---|-|---|-|-------->
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | +---+ +---+   +---------+   +---+ +---+ |
 *                    |  initialize                    close    |
 *                    +-----------------------------------------+
 * </pre>
```
`Transaction`就像是一个“壳子”，它首先会将目标函数用`wrapper`（一组`initialize`及`close`方法称为一个`wrapper`） 封装起来，同时需要使用`Transaction`类暴露的`perform`方法去执行它。如上面的注释所示，在`anyMethod`执行之前，`perform`会先执行所有 `wrapper`的`initialize`方法，执行完后，再执行所有`wrapper`的`close`方法。这就是`React`中的事务机制。

结合我们刚才的点击事件，事件其实是作为一个`callback`回调函数在事务中调用的，调用之前，批量更新策略事务会把`isBatchingUpdates`置为`true`，然后执行`callback`方法，执行完毕之后，把`isBatchingUpdates`置为`false`，然后再循环所有的`dirtyComponents`调用`updateComponent`更新组件。

所以刚才的点击事件，其实可以这样理解
```js
add = () => {
  // 进来先锁上
  isBatchingUpdates = true
  console.log('add前', this.state.num)
  this.setState({
    num: this.state.num + 1
  });
  console.log('add后', this.state.num)
  // 执行完函数再放开
  isBatchingUpdates = false
}
```
这种情况下，`setState`是异步的。

我们再来看看`setTimeout`的情况
```js
reduce = () => {
  // 进来先锁上
  isBatchingUpdates = true
  setTimeout(() => {
    console.log('reduce前的', this.state.num)
    this.setState({
      num: this.state.num - 1
    });
    console.log('reduce后的', this.state.num)
  },0);
  // 执行完函数再放开
  isBatchingUpdates = false
}
```
因为`setTimeout`是在之后的宏任务中执行的，所以这时候运行的`setState`，`isBatchingUpdates`已经被置为`false`了，它会立即执行更新，所以具备了同步的特性。**setState 并不是具备同步这种特性，只是在某些特殊的执行顺序下，脱离了异步的控制**

## 总结
`setState`并不是单纯同步/异步的，它的表现会因调用场景的不同而不同：在`React`钩子函数及合成事件中，它表现为异步；而在 `setTimeout`、`setInterval`等函数中，包括在`DOM`原生事件中，它都表现为同步。这种差异，本质上是`React`事务机制和批量更新机制的工作方式来决定的。
