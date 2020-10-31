---
# 主题使用方法：https://github.com/xitu/juejin-markdown-themes
theme: juejin
highlight: github
---
## 前言
换了新公司，工作中使用的技术栈也从`Vue`换到了`React`，作为一个`React`新人，经常的总结和思考才能更快更好的了解这个框架。这里分享一下我这两个月来使用`React`总结的一些性能优化的方法。

因为目前公司的项目是全面拥抱`hooks`的，所以只会涉及`function`组件写法，不包含`class`组件写法的相关内容。**注意：本文只涉及到一些业务开发层面的代码优化，很多通用的优化思想，比如虚拟列表，图片懒加载，节流防抖，webpack优化等等内容都不会涉及到。**

## React的更新机制

要来优化代码，首先我们来简单了解一下`React`的更新机制。看下图


![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11b82f66bd5f443ab6c2d079dbbb6ff0~tplv-k3u1fbpfcp-zoom-1.image)


我们重点来看第一步到第二步这个过程，当一个组件的`props`或`state`改变的时候，当前组件会重新`render`。当前组件下面的所有子、孙、曾孙。。。组件不管是否用到了父组件的`props`，全都会重新`render`。这是一个跟`Vue`很大的区别，`Vue`中所有组件的渲染都是由各自的`state`控制，父组件跟子组件的渲染都是独立的。

性能优化，主要是三块内容，

- **提高diff算法的dom重复利用率**
- **减少资源的加载**
- **减少组件的render次数和计算量（最重点的一块）**

## 遍历列表使用key

这个跟`React`的`diff`算法有关，是一个很简单，可以作为必须遵守规范的一个优化。

在所有的需要遍历的列表当中，都加上一个`key`值，这个值不能是那种遍历时候的序号，必须是一个固定值。比如该条数据`id`。

这个`key`可以帮助diff算法更好的复用`dom`元素，而不是销毁再重新生成。


## 精简不必要的节点
因为`React`的`diff`算法跟`Vue`一样是对于虚拟`dom`从父到子，一层层同级的比较。所以减少节点的嵌套，可以有效的减少`diff`算法的计算量。
```html
<div className="root">
  <div>
    <h1>我的名字：{name}</h1>
  </div>
  <div>
    <p>我的简介: {content}</p>
  </div>
</div>
// 完全可以精简为
<div className="root">
  <h1>我的名字：{name}</h1>
  <p>我的简介: {content}</p>
</div>
```
## 精简state

不需要把所有状态都放在组件的`state`中，只有那些需要响应式的数据才应该存入`state`。

## 不要使用CSS内联样式
在`React`中处理样式有三种
- css Module
- css in js（以styled-components为代表的）
- 内联css （把样式写在组件的style里）

对于`css Module`和`css in js`来说，其实都有优缺点，用哪个其实都没问题。虽然很多人说`css Module`性能要比`css in js`好，但是那点性能真的不值一提。

这边要说的是`内联css`，如果你没有那种必须通过控制`style`来修改组件内容或者样式的需求的话，千万不要写。

这块在后面`render`的优化中会细讲。

## 使用useMemo减少重复计算

来看一个例子
```jsx
import React from 'react';

export default function App() {
  const [num, setNum] = useState(0);
  
  const [factorializeNum, setFactorializeNum] = useState(5);

  // 阶乘函数
  const factorialize = (): Number => {
    console.log('触发了');
    let result = 1;
    for (let i = 1; i <= factorializeNum; i++) {
      result *= i;
    }
    return result;
  };

  return (
    <>
      {num}
      <button onClick={() => setNum(num + factorialize())}>修改num</button>
      <button onClick={() => setFactorializeNum(factorializeNum + 1)}>修改阶乘参数</button>
    </>
  );
}
```
在这个组件里，每次点击`修改num`这个按钮，都会打印一次`触发了`，阶乘函数会重新计算一遍。但是其实参数是没有变化的，返回的结果也是没有变化的。

我们可以使用`useMemo`来缓存计算结果，避免重复计算。

```jsx
import React, { useMemo } from 'react';

export default function App() {
  const [num, setNum] = useState(0);
  
  const [factorializeNum, setFactorializeNum] = useState(5);

  // 当factorializeNum值不变的时候，这个函数不会再重复触发了
  const factorialize = useMemo((): Number => {
    console.log('触发了');
    let result = 1;
    for (let i = 1; i <= factorializeNum; i++) {
      result *= i;
    }
    return result;
  }, [factorializeNum]);

  return (
    <>
      {num}
      <button onClick={() => setNum(num + factorialize())}>修改num</button>
      <button onClick={() => setFactorializeNum(factorializeNum + 1)}>修改阶乘参数</button>
    </>
  );
}
```
## 多用三元表达式
我们写一些组件的时候经常会碰到这种需求，根据参数的不同，渲染不同的组件。例
```js
const App = () => {
  const [type, setType] = useState(1);

  if (type === 1) {
    return (
      <>
        <Component1>component1</Component1>
        <Component2>component2</Component2>
        <Component3>component3</Component3>
      </>
    );
  }

  return (
    <Component2>component2</Component2>
    <Component3>component3</Component3>
  );
};
```
上面的代码乍一看其实没啥问题，根据类型的不同，返回不同的组件。但是对于`diff`算法来说，它会对同级的新旧节点进行比较，当类型变化的时候，`Component1`没有生成了，对于`diff`算法来说，他会拿旧的第一项`Component1`跟新的第一项`Component2`比较，因为没有`key`，而且这是组件, `diff`算法会深入到组件的子元素中再去同级比较。假设这三个组件都是不一样的，`diff`算法就会把旧节点的三个组件全部销毁，再重新生成两个新组件。

但是按性能来说，其实只需要销毁第一个组件，复用剩下的那两个就可以。

加`key`当然可以，但是我们可以使用更简单的三元表达式。
```html
<>
  {type === 1 && <Component1>component1</Component1>}
  <Component2>component2</Component2>
  <Component3>component3</Component3>
</>
```
当类型不符合的时候，三元表达式会放置一个`null`，`diff`算法会拿这个`null`跟旧的`component1`进行比较，剩下的两个组件顺序不变，`diff`算法会进行复用。而且这种方式，代码也更加精简。

## 异步组件（懒加载组件）

最典型场景是`tab`页面切换，当`tab`切换到相应的页面上时，再去加载相应页面的组件js。

这些的组件资源不会包含在主包里，在后续在用户需要的时候，再去加载相关的组件js资源。可以提高页面的加载速度，减少无效资源的加载。

主要用到两个方法`React.Suspense`和`React.lazy`
```js
import React from 'react';

export default (props) => {
  return (
    <>
      <Drawer>
        <Tabs defaultActiveKey="1">
          <TabPane>
            <React.Suspense fallback={<Loading />}>
              {React.lazy(() => import('./Component1'))}
            </React.Suspense>
          </TabPane>
          <TabPane>
            <React.Suspense fallback={<Loading />}>
              {React.lazy(() => import('./Component2'))}
            </React.Suspense>
          </TabPane>
        </Tabs>
      </Drawer>
    </>
  );
};
```
使用上面的方法之后，`webpack`会把这个`import`的组件单独打包成一个`js`。在`tab`切换到相应的页面时，加载这个`js`，渲染出相应的组件。

## 减少组件的render（重点）

### 使用React.memo
我们先来看个例子
```jsx
import React from 'react';

const Child = () => {
  console.log('触发Child组件渲染');
  return (
    <h1>这是child组件的渲染内容！</h1>
  )
};

export default () => {
  const [num, setNum] = useState(0);
  
  return (
    <>
      {num}
      <button onClick={() => setNum(num + 1)}>num加1</button>
      <Child />
    </>
  );
}
```
当我们每次点击`num加1`这个按钮的时候，我们都会在控制台发现打印了一次`触发Child组件渲染`。说明`Child`这个组件在我们父组件的`state`变化之后，每次都会重新`render`。

我们可以使用`React.memo`来避免子组件的重复`render`。
```jsx
import React from 'react';

const Child = React.memo(() => {
  console.log('触发Child组件渲染');
  return (
    <h1>这是child组件的渲染内容！</h1>
  )
});

export default () => {
  const [num, setNum] = useState(0);
  
  return (
    <>
      {num}
      <button onClick={() => setNum(num + 1)}>num加1</button>
      <Child />
    </>
  );
}
```
`React.memo`会判断子组件的`props`是否有改变，如果没有，将不会重复`render`。这时候我们点击`num加1`按钮，`Child`将不会重复渲染。

### 不要直接使用内联对象
我们再来看一个例子
```jsx
import React from 'react';

const Child = React.memo((props) => {
  const { style } = props;
  console.log('触发Child组件渲染');
  return (
    <h1 style={style}>这是child组件的渲染内容！</h1>
  )
});

export default () => {
  const [num, setNum] = useState(0);
  
  return (
    <>
      {num}
      <button onClick={() => setNum(num + 1)}>num加1</button>
      <Child style={{color: 'green'}}/>
    </>
  );
}
```
这个相比较上一个例子，就是给`Child`组件多传入了一个`style`参数。传入的参数是一个静态的对象，你觉得现在子组件会重复渲染吗？

一开始我觉得不会，实际测试下来，发现子组件又开始了重复渲染。

`state`改变，父组件重新`render`的时候，像这种`{color: 'green'}`会重新生成，这个对象的内存地址会变成一个新的。而`React.memo`只会对`props`进行浅层的比较，因为传入对象的内存地址修改了，所以`React.memo`就以为传入的`props`有新的修改，就重新渲染了子组件。

我们可以有两种方式来修改。
```jsx
// 如果传入的参数是完全独立的，没有任何的耦合
// 可以将该参数，提取到渲染函数之外
const childStyle = { color: 'green' };
export default () => {
  const [num, setNum] = useState(0);
  
  return (
    <>
      {num}
      <button onClick={() => setNum(num + 1)}>num加1</button>
      <Child style={childStyle}/>
    </>
  );
}
// 如果传入的参数需要使用渲染函数里的参数或者方法
// 可以使用useMemo
export default () => {
  const [num, setNum] = useState(0);
  const [style, setStyle] = useState('green');
  // 如果不需要参数
  const childStyle = useMemo(() => ({ color: 'green' }), []);
  // 如果需要使用state或者方法
  const childStyle = useMemo(() => ({ color: style }), [style]);
  return (
    <>
      {num}
      <button onClick={() => setNum(num + 1)}>num加1</button>
      <Child style={childStyle}/>
    </>
  );
}
```

### 传入组件的函数使用React.useCallback

函数导致子组件重新渲染的原理跟上面的内联对象一样，也是因为父组件的重新渲染，导致函数方法的内存地址发生变化，所以`React.memo`会认为`props`有变化，导致子组件重复渲染。

我们可以使用`React.useCallback`来缓存函数方法，避免子组件的重复渲染。
```jsx
export default () => {
  const [num, setNum] = useState(0);
  const oneFnc = useCallback(() => {
    console.log('这是传入child的方法');
  }, []);
  return (
    <>
      {num}
      <button onClick={() => setNum(num + 1)}>num加1</button>
      <Child onFnc={oneFnc} />
    </>
  );
}
```

同理，要避免在子组件的传入参数上直接写匿名函数。
```html
// 不要直接写匿名函数
<Child onFnc={() => console.log('这是传入child的方法')} />
```

### 使用children来避免React Context子组件的重复渲染

对于我们常用的`Context`，我们不但可以使用`React.Memo`来避免子组件的重复渲染，我们还可以通过`children`的方式。
```js
import React, { useContext, useState } from 'react';

const DemoContext = React.createContext();

const Child = () => {
  console.log('触发Child组件渲染');
  return (
    <h1 style={style}>这是child组件的渲染内容！</h1>
  )
};

export default () => {
  const [num, setNum] = useState(0);
  return (
    <DemoContext.Provider value={num}>
      <button onClick={() => setNum(num + 1)}>num加1</button>
      <Child />
      {...一些其他需要使用num参数的组件}
    </DemoContext.Provider>
  );
}
```

在这里可以使用`children`方法来避免`Child`的重复渲染。

```js
import React, { useContext, useState } from 'react';

const DemoContext = React.createContext();

const Child = () => {
  console.log('触发Child组件渲染');
  return (
    <h1 style={style}>这是child组件的渲染内容！</h1>
  )
};

function DemoComponent(props) {
  const { children } = props;
  const [num, setNum] = useState(0);
  return (
    <DemoContext.Provider value={num}>
      <button onClick={() => setNum(num + 1)}>num加1</button>
      {children}
    </DemoContext.Provider>
  );
}

export default () => {
  return (
    <DemoComponent>
      <Child />
      {...一些其他需要使用num参数的组件}
    </DemoComponent>
  );
}
```

这时候，修改`state`,只是对于`DemoComponent`这个组件内部进行`render`，对于外部传入的`Child`组件，将不会重复渲染。

## 总结

上面这些都是我平时开发当中真实碰到过的问题，相信也是所有`React`开发者都会碰到的问题，涉及到的技术不深，希望给一些新入坑`React`的同学有所帮助。

## 感谢

谢谢大家的阅读，如果觉得对你有所帮助，请帮忙点个赞支持一下！

