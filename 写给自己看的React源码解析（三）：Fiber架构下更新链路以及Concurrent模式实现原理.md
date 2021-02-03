
## 前言
本文内容涉及到很多渲染链路中的原理以及源码方法，所以在看本文之前，需要对于`React`的`render`渲染流程有大致的了解。不清楚的同学可以先看我的第一篇源码解析文章。

[写给自己看的React源码解析（一）：你的React代码是怎么渲染成DOM的？
](https://juejin.cn/post/6915398787292725261#heading-21)

本文主要解析`fiber`架构更新链路的`双缓冲`模式以及`Concurrent`模式下`时间切片`，`优先级`的实现原理。

## 双缓冲
双缓冲模式的主要用处，是**能够帮我们较大限度地实现`Fiber`节点的复用，减少性能方面的开销。**

在之前的文章中，我们知道在首次渲染的时候会创建出两颗树，`current`树与`workInProgress`树。`current`树与`workInProgress`树，其实就是两套缓冲数据：当`current`树被渲染到页面上时，所有的数据更新都会由`workInProgress`树来承接。`workInProgress`树将会在内存里悄悄地完成所有改变，直到下次进行渲染的`commit`阶段执行完毕之后,`fiberRoot`对象的`current`会指向`workInProgress`树，`workInProgress`树就会变成渲染到页面上的`current`树。

我们用一个实际例子来帮助理解：
```jsx
import { useState } from 'react';
function App() {
  const [state, setState] = useState(0)
  return (
    <div className="App">
      <div onClick={() => { setState(state + 1) }}>
        <p>{state}</p>
      </div>
    </div>
  );
}
```
### 初始化
这个例子的功能很简单，就是点击一次，数字加1。上面的demo在`render`阶段结束后，`commit`阶段结束前的两颗`fiber`树如下图所示

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3021e1bacb7840b4a584993e851b76f9~tplv-k3u1fbpfcp-watermark.image)

`commit`阶段完成，`workInProgress`树被渲染到页面上，这时候`fiberRoot`对象的`current`会指向`workInProgress`树，这个当前被渲染的`fiber`树。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7969ceb670fe4b779e34902ab056a3e5~tplv-k3u1fbpfcp-watermark.image)

### 第一次更新
点击一次数字，我们进入第一次的更新流程。重点看`beginWork`调用链路中的`createWorkInProgress`方法。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eed940c8f7774cdda48ad24b9295c420~tplv-k3u1fbpfcp-watermark.image)

上图中，`workInProgress`树下面的子节点的`current.alternate`对应的就是`current`树的子节点，但是`current`树目前没有子节点，所以为null，进入等于null的流程。按照`workInProgress`的子节点的属性给`current`树创建出相同的子节点。

然后在`commit`阶段结束后，`current`树会被渲染到页面上，`fiberRoot`对象的`current`会指回到`current`树，具体如下图
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bd0e5acd5aa44607abd5a1e3abc9db4d~tplv-k3u1fbpfcp-watermark.image)

### 第二次更新
再点击一次数字，触发state的第二次更新，还是看之前的`createWorkInProgress`方法。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eed940c8f7774cdda48ad24b9295c420~tplv-k3u1fbpfcp-watermark.image)

这时候，因为两颗树都已经构建完成，所以`current.alternate`是存在的。所以之后每次通过`beginWork` 触发`createWorkInProgress`调用时，都会一致地走入`else`里面的逻辑，也就是直接复用现成的节点。
这也就是双缓冲机制实现节点复用的方法。

## 更新链路要素
`React`源码解析第一篇分析了首次渲染的链路，更新的链路其实跟首次渲染大致一样。

首次渲染可以理解为一种特殊的更新，`ReactDOM.render`,`setState`,`useState`一样，都是一种触发更新的姿势。这些方法发起的调用链路很相似，是因为它们最后“殊途同归”，都会通过创建`update`对象来进入同一套更新工作流。

按demo的流程来，点击数字之后，会触发一个`dispatchAction`方法，在该方法中，会完成`update`对象的创建

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/597f50a1865b4c72acdbca578bff3b22~tplv-k3u1fbpfcp-watermark.image)

`update`创建完成之后，会跟首次渲染一样，进入`updateContainer`方法(首次渲染链路中的`update`会在这个方法里创建)，这里主要是两个方法
```js
enqueueUpdate(current, update);
scheduleUpdateOnFiber(current, lane, eventTime);
```
- `enqueueUpdate`：将`update`入队。每一个`Fiber`节点都会有一个属于它自己的`updateQueue`，用于存储多个更新，这个`updateQueue`是以链表的形式存在的。在`render`阶段，`updateQueue`的内容会成为 `render`阶段计算`Fiber`节点的新`state`的依据。

- `scheduleUpdateOnFiber`：调度`update`。这个方法后面紧跟的就是`performSyncWorkOnRoot`所触发的`render`阶段。

这里有一个点需要提示一下：`dispatchAction`中，调度的是当前触发更新的节点，这一点和挂载过程需要区分开来。在挂载过程中，`updateContainer`会直接调度根节点。其实，对于更新这种场景来说，大部分的更新动作确实都不是由根节点触发的，而`render`阶段的起点则是根节点。所以在`scheduleUpdateOnFiber `中，有这样一个方法

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7e2c3cc385f548c39afac2f3a758789a~tplv-k3u1fbpfcp-watermark.image)

它会从当前`Fiber`节点开始，向上遍历直至根节点，并将根节点返回。所以，我们说`React`的更新流程，是从根节点开始，重新遍历整个`fiber`树，这也是为什么我们平时的性能优化的重点都在减少组件的重新`render`上。

在`scheduleUpdateOnFiber`中，还有一个重要的判断，那就是对于同步和异步的判断逻辑。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ecb4d7316d034e79ab8e685741aef358~tplv-k3u1fbpfcp-watermark.image)

之前我们分析同步的首次渲染流程的时候，走的是`performSyncWorkOnRoot`方法，但是对于异步模式，会运行`ensureRootIsScheduled`方法。来看下一段核心逻辑

```js
if (newCallbackPriority === SyncLanePriority) {
    // 同步更新的 render 入口
    newCallbackNode = scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));
  } else {
    // 将当前任务的 lane 优先级转换为 scheduler 可理解的优先级
    var schedulerPriorityLevel = lanePriorityToSchedulerPriority(newCallbackPriority);
    // 异步更新的 render 入口
    newCallbackNode = scheduleCallback(schedulerPriorityLevel, performConcurrentWorkOnRoot.bind(null, root));
  }
```

从这段逻辑中我们可以看出，`React`会以当前更新任务的优先级类型为依据，决定接下来是调度 `performSyncWorkOnRoot`还是`performConcurrentWorkOnRoot`。这里调度任务用到的函数分别是 `scheduleSyncCallback`和`scheduleCallback`，这两个函数在内部都是通过调用 `unstable_scheduleCallback`方法来执行任务调度的。这个方法是`Scheduler`（调度器）中导出的一个核心方法。

`Scheduler`的核心能力，就是让`fiber`架构实现了`时间切片`与`优先级调度`这两个核心特征。

## 时间切片
先来了解一下时间切片到底是做了什么事情？
```js
import React from 'react';
function App() {
  const arr = new Array(1000).fill(0);
  return (
    <div className="App">
      <div className="container">
        {
          arr.map((i, index) => <p>{`测试文本第${index}行`}</p>)
        }
      </div>
    </div>
  );
}
```
上面的代码就是渲染1000条`p`标签到页面上，当我们使用`ReactDOM.render`进行渲染，因为它是一个同步的过程，所有的链路都会在一个宏任务里执行掉。根据不同用户电脑和浏览器的性能不同，这个宏任务的执行时间，可能是100ms、200ms、300ms甚至更多。因为`js`线程和渲染线程是互斥的，在执行这个比较长时间的宏任务时，我们浏览器的渲染线程将被阻塞。我们知道浏览器的刷新频率为`60Hz`也就是说每`16.6ms`就会刷新一次,这种长时间的宏任务导致的渲染线程阻塞，将会产生明显的卡顿、掉帧。

而时间切片，就是把这段需要较长时间运行的宏任务“切”开，变成一段段尽量保证运行时间在浏览器刷新间隔时间之下的宏任务。给渲染线程留出时间，保证渲染的流畅度。我们来看两张图，第一张是同步模式下的调用栈

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5cb790365b4e45a3b3fbecfc1229a1c4~tplv-k3u1fbpfcp-watermark.image)

下一张是把`ReactDOM.render`调用改为`createRoot`，用`Concurrent`(异步)模式来进行渲染

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bd0df6d5652d4f2292e9edb60dd543a0~tplv-k3u1fbpfcp-watermark.image)

我们可以看到，本来一个长时间的“大任务”被切成了一个个短时间的“小任务”。

### 时间切片是如何实现的？
根据上文对`scheduleUpdateOnFiber`的分析，在同步的模式下，`React`会调用`performSyncWorkOnRoot`，在这个链路下，会通过`workLoopSync`方法来循环创建`Fiber`节点、构建`Fiber`树。
```js
function workLoopSync() {
  // 若 workInProgress 不为空
  while (workInProgress !== null) {
    // 针对它执行 performUnitOfWork 方法
    performUnitOfWork(workInProgress);
  }
}
```
这是一个无法中断的过程，开始了就无法停止。

而在异步的模式下,`React`会调用`performConcurrentWorkOnRoot`，通过`renderRootConcurrent`调用 `workLoopConcurrent`来构建`Fiber`树。

```js
function workLoopConcurrent() {
  // Perform work until Scheduler asks us to yield
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

我们可以发现，异步的方法里，其实就只是多了一个`shouldYield()`方法，当`shouldYield()`为`true`的时候，while循环将停止，将主线程让给渲染线程。

`shouldYield`的本体其实也是调度器里导出的一个方法`Scheduler.unstable_shouldYield`，方法很简单。[源码地址](https://github.com/facebook/react/blob/27659559ebfd6b7119bfc0ff02ecb851c135020c/packages/scheduler/src/forks/SchedulerPostTask.js#L62)

```js
export function unstable_shouldYield() {
  return getCurrentTime() >= deadline;
}
```
就是当当前时间大于`deadline`这个当前时间切片的到期时间时，就返回`true`，停止`workLoopConcurrent`循环。

我们来看下`deadline`是怎么定义的
```js
deadline = getCurrentTime() + yieldInterval;
```
`getCurrentTime()`就是当前时间，而`yieldInterval`是一个常量,5ms，[源码地址](https://github.com/facebook/react/blob/27659559ebfd6b7119bfc0ff02ecb851c135020c/packages/scheduler/src/forks/SchedulerPostTask.js#L55)

```js
const yieldInterval = 5;
```
所以说，时间切片的间隔是`5ms`(实际应该都是比`5ms`稍大，因为必须等当前的`fiber`节点构建完成之后，才会通过`shouldYield()`方法判断是否到期)

当`workLoopConcurrent`循环中断之后，`React`会重新发起调度（`setTimeout`或者`MessageChannel`方式），检查是否存在事件响应、更高优先级任务或其他代码需要执行，如果有则执行，如果没有则重新创建工作循环`workLoopConcurrent`，执行剩下的工作中`Fiber`节点构建。

## 优先级调度
在更新链路中，无论是`scheduleSyncCallback`还是`scheduleCallback`，最终都是通过调用 `unstable_scheduleCallback`来发起调度的。
`unstable_scheduleCallback`是`Scheduler`导出的一个核心方法，它将结合任务的优先级信息为其执行不同的调度逻辑。[源码地址](https://github.com/facebook/react/blob/v17.0.1/packages/scheduler/src/Scheduler.js#L279)

```js
function unstable_scheduleCallback(priorityLevel, callback, options) {
  // 获取当前时间
  var currentTime = getCurrentTime();
  // 声明 startTime，startTime 是任务的预期开始时间
  var startTime;
  // 以下是对 options 入参的处理
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    // 若入参规定了延迟时间，则累加延迟时间
    if (typeof delay === 'number' && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
  } else {
    startTime = currentTime;
  }
  // timeout 是 expirationTime 的计算依据
  var timeout;
  // 根据 priorityLevel，确定 timeout 的值
  switch (priorityLevel) {
    case ImmediatePriority:
      timeout = IMMEDIATE_PRIORITY_TIMEOUT;
      break;
    case UserBlockingPriority:
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT;
      break;
    case IdlePriority:
      timeout = IDLE_PRIORITY_TIMEOUT;
      break;
    case LowPriority:
      timeout = LOW_PRIORITY_TIMEOUT;
      break;
    case NormalPriority:
    default:
      timeout = NORMAL_PRIORITY_TIMEOUT;
      break;
  }
  // 优先级越高，timout 越小，expirationTime 越小
  var expirationTime = startTime + timeout;
  // 创建 task 对象
  var newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    sortIndex: -1,
  };
  if (enableProfiling) {
    newTask.isQueued = false;
  }
  // 若当前时间小于开始时间，说明该任务可延时执行(未过期）
  if (startTime > currentTime) {
    // 将未过期任务推入 "timerQueue"
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);
    
    // 若 taskQueue 中没有可执行的任务，而当前任务又是 timerQueue 中的第一个任务
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      // All tasks are delayed, and this is the task with the earliest delay.
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // 那么就派发一个延时任务，这个延时任务用于检查当前任务是否过期
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    // else 里处理的是当前时间大于 startTime 的情况，说明这个任务已过期
    newTask.sortIndex = expirationTime;
    // 过期的任务会被推入 taskQueue
    push(taskQueue, newTask);
    
    ......
    
    // 执行 taskQueue 中的任务
    requestHostCallback(flushWork);
  }
  return newTask;
}
```
`unstable_scheduleCallback`的主要工作是针对当前任务创建一个`task`，然后结合`startTime`信息将这个`task`推入`timerQueue`或`taskQueue`，最后根据`timerQueue`和`taskQueue`的情况，执行延时任务或即时任务。

这里需要知道几个概念
- startTime：任务的开始时间。
- expirationTime：这是一个和优先级相关的值，expirationTime 越小，任务的优先级就越高。
- timerQueue：一个以 startTime 为排序依据的小顶堆，它存储的是 startTime 大于当前时间（也就是待执行）的任务。
- taskQueue：一个以 expirationTime 为排序依据的小顶堆，它存储的是 startTime 小于当前时间（也就是已过期）的任务。

堆是一种特殊的完全二叉树。如果对一棵完全二叉树来说，它每个结点的结点值都不大于其左右孩子的结点值，这样的完全二叉树就叫“小顶堆”。小顶堆自身特有的插入和删除逻辑，决定了无论我们怎么增删小顶堆的元素，其根节点一定是所有元素中值最小的一个节点。

我们来看下核心逻辑
```js
if (startTime > currentTime) {
    // 将未过期任务推入 "timerQueue"
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);
    
    // 若 taskQueue 中没有可执行的任务，而当前任务又是 timerQueue 中的第一个任务
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      ...... 
      
      // 那么就派发一个延时任务，这个延时任务用于检查当前任务是否过期
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    // else 里处理的是当前时间大于 startTime 的情况，说明这个任务已过期
    newTask.sortIndex = expirationTime;
    // 过期的任务会被推入 taskQueue
    push(taskQueue, newTask);
    
    ......
    
    // 执行 taskQueue 中的任务
    requestHostCallback(flushWork);
  }
```
若判断当前任务是未过期任务，那么该任务会在`sortIndex`属性被赋值为`startTime`后，被推入`timerQueue`。`taskQueue`里存储的是已过期的任务，`peek(taskQueue) `取出的任务若为空，则说明`taskQueue`为空、当前并没有已过期任务。在没有已过期任务的情况下，若当前任务（`newTask`）就是`timerQueue`中需要最早被执行的未过期任务，那么`unstable_scheduleCallback`会通过调用`requestHostTimeout`，为当前任务发起一个延时调用。

注意，这个延时调用（也就是`handleTimeout`）并不会直接调度执行当前任务——它的作用是在当前任务到期后，将其从 `timerQueue`中取出，加入`taskQueue`中，然后触发对`flushWork`的调用。真正的调度执行过程是在`flushWork`中进行的。`flushWork`中将调用`workLoop`，`workLoop`会逐一执行`taskQueue`中的任务，直到调度过程被暂停（时间片用尽，将重新发起`Task`调度）或任务全部被清空。

当下`React`发起`Task`调度的姿势有两个：`setTimeout`、`MessageChannel`。在宿主环境不支持`MessageChannel`的情况下，会降级到`setTimeout`。但不管是`setTimeout`还是`MessageChannel`，它们发起的都是异步任务（宏任务，将在下次`eventLoop`中被调用）。

## 感谢
如果本文对你有所帮助，请帮忙点个赞，感谢！
