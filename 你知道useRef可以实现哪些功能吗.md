---
# 主题列表：juejin, github, smartblue, cyanosis, channing-cyan
# 贡献主题：https://github.com/xitu/juejin-markdown-themes
theme: smartblue
highlight: vs2015
---

## 前言

之前其实对于`useRef`使用的并不多，但在最近公司的新项目开发中，碰到了许多需要使用`useRef`来解决的业务场景，顺便也对`useRef`好好的研究了一番，以此文作为学习的总结。

注意：本文只介绍现在`hooks`中的`useRef`、`forwardRef`以及`useImperativeHandle`。`Class`时期的`createRef`不在本文内容之内。
## useRef

首先我们需要知道，`useRef`到底是什么？它可以用来做什么？

**`useRef`主要的功能就是帮助我们获取到DOM元素或者组件实例，它还可以保存在组件生命周期内不会变化的值。**

来看一个`React`官网上的例子

```js
function TextInputWithFocusButton() {
  const inputEl = useRef(null);
  const onButtonClick = () => {
    // `current` 指向已挂载到 DOM 上的文本输入元素
    inputEl.current.focus();
  };
  return (
    <>
      <input ref={inputEl} type="text" />
      <button onClick={onButtonClick}>Focus the input</button>
    </>
  );
}
```
在这个例子里，`ref`会被分配给元素DOM元素，我们可以通过`inputEl.current`获取到这个input的DOM对象，并使用它完成我们需要的功能。

这里需要注意，`ref`有两个特点
- 1.每次渲染`useRef`的返回值都是同一个（相同的引用）
- 2.`ref.current`发生变化的时候，他不像其他的一些`hook`，它不会造成组件的重新渲染

根据这两个特点，我们可以知道。

**1.`ref`如果获取的是DOM或者组件实例，请尽量拿`ref.current`不要作为其他的hooks的依赖项**

这里的不要作为依赖项不是因为程序会报错，而是因为`ref`的特性，在获取的是DOM或者组件实例的场景下，作为依赖项，跟普通的`state`表现不太一致。非常容易导致代码的业务逻辑出现`bug`。

首先，不管哪种情况，都不要将`ref`作为依赖项，因为`ref`一直是同一个引用，所以不会触发依赖变化。

```js
useEffect(()=> {
  // ...
}, [ref])
// 等同于
useEffect(()=> {
  // ...
}, [])
```
对于`ref.current`来说，需要区分一下情况，**如果你只是把`useRef`作为一个保存数据的功能**，使用`ref.current`没有问题，本文下面的一个深比较依赖的`hooks`就是用到了`ref.current`作为依赖。

但是如果**你是拿`useRef`来获取DOM或者组件实例**，`ref.current`的依赖项除了初始化触发一次，在组件更新的时候会触发一次之后，后续组件的重新渲染将都不会触发这个`ref.current`的依赖项。（组件更新还会触发一次是因为`ref.current`在初始化的时候，从`null`到组件的`ref`挂载完成，`ref.current`会有一个数据变化）

**2.如果需要手动去修改`ref.current`的值，最好不要在`render`里直接写，因为组件的`render`更新可能会有多次，会导致值的多次变更。最好写在`useEffect`方法里。**

**3.如果你需要在组件`ref`挂载完成之后调用一个你自己的方法，你需要使用[callback ref](https://zh-hans.reactjs.org/docs/hooks-faq.html#how-can-i-measure-a-dom-node)**


这是官网的一个demo。
```js
function MeasureExample() {
  const [height, setHeight] = useState(0);

  const measuredRef = useCallback(node => {
    if (node !== null) {
      setHeight(node.getBoundingClientRect().height);
    }
  }, []);

  return (
    <>
      <h1 ref={measuredRef}>Hello, world</h1>
      <h2>The above header is {Math.round(height)}px tall</h2>
    </>
  );
}
```


那如果要获取组件的实例呢？在函数式组件中，我们不能直接去给它添加`ref`属性，我们需要使用`forwardRef`来包裹函数式组件。

## forwardRef
我们用`forwardRef`包裹函数式组件，见下例
```ts
const Parent = () => {
  const childRef = useRef(null);

  useEffect(() => {
    ref.current.focus();
  }, []);

  return (
    <>
      <Child ref={ref} />
    </>
  );
};
const Child = forwardRef((props, ref) => {
  return <input type="text" name="child" ref={ref} />;
});
```
使用`forwardRef`包裹之后，函数式组件会获得被分配给自己的ref（作为第二个参数）。如果你没有使用`forwardRef`而直接去`ref`的话，`React`会报错。

## useImperativeHandle
上面`forwardRef`的例子中，`Parent`中的`ref`拿到了`Child`组件的完整实例，它不但可以使用`focus`方法，还可以使用其它所有的DOM方法，比如`blur`,`style`。这种方式是不推荐的，我们需要严格的控制`ref`的权力，控制它所能调用到的方法。

所以我们要使用`useImperativeHandle`来限制暴露给父组件的方法。
```ts
const Parent = () => {
  const childRef = useRef(null);

  useEffect(() => {
    // 这里只能调用到focus方法
    ref.current.focus();
  }, []);

  return (
    <>
      <Child ref={ref} />
    </>
  );
};
const Child = forwardRef((props, ref) => {
  const inputRef = useRef(null);
  useImperativeHandle(ref, () => ({
    focus: () => {
      inputRef.current.focus();
    }
  }));
  return <input type="text" name="child" ref={inputRef} />;
});
```
这样子，我们就可以手动控制需要暴露给父组件的方法。

## 在开发中的实际应用

### 获取上一次的值

官方给的一个自定义hook
```js
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}
```
这个hooks返回出来的值，在渲染的过程中，总是会显示上一次的值。我们来解析一下这个函数的运行步骤。
假设上例中`ref`的初始值传入的`value`是0,每次数据更新传入的都是递增的数据，比如1，2，3。
- 1. 初始化，ref.current = 0，渲染出来。
- 2. 数据变化，`value`传入1, 因为`useEffect`会在渲染完毕之后才执行，所以这次的渲染过程中，这个为1的`value`值不会赋值给`ref.current`。渲染出来的还是上一个值0，渲染完毕了，`ref.current`变为1。但是`ref.current`变化不会触发组件的重新渲染，所以需要等到下次的渲染才能显示到页面上。
- 3. 如此往复，渲染的就总是上一次的值。

### 使用useRef来保存不需要变化的值
因为`useRef`的返回值在组件的每次`redner`之后都是同一个，所以它可以用来保存一些在组件整个生命周期都不需要变化的值。最常见的就是定时器的清除场景。

刚开始在`React`里写定时器，你可能会这样写
```js
const App = () => {
  let timer;

  useEffect(() => {
    timer = setInterval(() => {
      console.log('触发了');
    }, 1000);
  },[]);

  const clearTimer = () => {
    clearInterval(timer);
  }

  return (
    <>
      <Button onClick={clearTimer}>停止</Button>
    </>)
}
```
但是上面这个写法有个巨大的问题，如果这个`App`组件里有`state`变化或者他的父组件重新`render`等原因导致这个`App`组件重新`render`的时候，我们会发现，点击按钮停止，定时器依然会不断的在控制台打印，定时器清除事件无效了。

为什么呢？因为组件重新渲染之后，这里的`timer`以及`clearTimer `方法都会重新创建，`timer`已经不是定时器的变量了。

所以对于定时器，我们都会使用`useRef`来定义变量。
```js
const App = () => {
  const timer = useRef();

  useEffect(() => {
    timer.current = setInterval(() => {
      console.log('触发了');
    }, 1000);
  },[]);

  const clearTimer = () => {
    clearInterval(timer.current);
  }

  return (
    <>
      <Button onClick={clearTimer}>停止</Button>
    </>)
}
```

我们还可以通过`useRef`的这个特性来实现一个深度依赖对比的`useEffect`。

普通的`useEffect`只是一个浅比较的方法，如果我们依赖的`state`是一个对象，组件重新渲染，这个`state`对象的值没变，但是内存引用地址变化了，一样会触发`useEffect`的重新渲染。
```js
const createObj = () => ({
    name: 'zouwowo'
});
useEffect(() => {
  // 这个方法会无限循环
}, [createObj()]);
```
我们来使用`useRef`实现一个深度依赖对比的`useDeepEffect`
```js
import equal from 'fast-deep-equal';
export useDeepEffect = (callback, deps) => {
  const emitEffect = useRef(0);
  const prevDeps = useRef(deps);
  if (!equal(prevDeps.current, deps)) {
    // 当深比较不相等的时候，修改emitEffect.current的值，触发下面的useEffect更新
    emitEffect.current++;
  }
  prevDeps.current = deps;
  return useEffect(callback, [emitEffect.current]);
}
```
### 让父组件调用子组件的方法

这个其实是我之前`vue`开发时候的思想，`vue`的数据和方法虽然跟`React`一样也是单向的，从父组件传递方法到子组件，然后子组件进行调用。但是`vue`可以通过`ref`来获取到子组件的实例，然后直接调用子组件的方法。在`React`可以通过`useImperativeHandle`方法将子组件的方法传递到父组件。

下面是我在实际项目中开发的一个自己保存状态的`Modal`组件。
```js
// 版本
// "@types/react": "16.9.53",
// "react": "16.14.0",
// "typescript": "3.7.5"

// 这是modal组件
// demoModal.tsx

import React, {
  useCallback,
  forwardRef,
  useImperativeHandle
} from 'react';
import { useToggle } from 'ahooks';
import { Modal, Button } from 'antd';
// 这里的ModalRef应该作为一个通用的类型，这里只是作为例子，所以直接写了
interface ModalRef {
  toggle: (value?: boolean) => void
}

interface Props {
  onConfirm: (value: any) => void
}

export default React.memo(forwardRef<ModalRef, Props>((props, ref) => {
  // 这里的props必须要加!，不然解构类型会报错。我不清楚什么原因，如果有同学知道，可以在下面评论。
  const { onConfirm } = props!;
  // 数据状态由Modal组件自己维护
  const [visible, { toggle }] = useToggle<boolean>(false);
  // 这里可以不定义类型，useModalToggle会从ref里推导出类型
  // 暴露出一个toggle方法给父组件去调用
  useImperativeHandle(ref, () => ({ toggle }));

  // 定义关闭页面的方法，为什么用useCallback可以看我的上一篇关于性能优化的文章
  const closeModal = useCallback(() => {
    toggle(false);
  }, []);

  const confirmHandle = useCallback(() => {
    onConfirm('传递参数给父组件');
  }, [onConfirm]);

  return (
    <Modal
      visible={visible}
      title="弹框标题"
      onCancel={closeModal}
      footer={[
        <Button type="primary" key="submit" onClick={confirmHandle}>确定</Button>,
        <Button key="reset" onClick={closeModal}>取消</Button>
      ]}
    >
      这是弹框内容
    </Modal>
  );
}));
```

```js
// 这是使用modal的组件
// App.ts

import React, { useCallback, useRef } from 'react';
import DemoModal from './DemoModal';

interface ModalRef {
  toggle: (value?: boolean) => void
}

export default () => {
  // 这里默认需要加一个null，不然在组件的ref使用会报类型错误
  const demoModalRef = useRef<ModalRef>(null);

  const modalConfirm = useCallback((val: string) => {
    console.log(val);
  }, []);

 // 下面的demoModalRef.current需要加一个？，因为初始化的时候current还未定义，创建ref之后才会生成。
 // 这里用到的toggle，就是子组件传递过来的方法
  return (
    <>
      <div onClick={() => demoModalRef.current?.toggle(true)}>点击打开弹框</div>
      <DemoModal
        ref={demoModalRef}
        onConfirm={modalConfirm}
      />
    </>
  );
};
```

为了更加方便的使用，我还把创建和使用`ref`的逻辑抽离出来作为两个hooks去使用。
```js
// useModalToggle.ts
import { useImperativeHandle, Ref, useCallback } from 'react';
import { useToggle } from 'ahooks';

const useModalToggle = <T>(ref: Ref<T>) => {
  const [visible, { toggle }] = useToggle<boolean>(false);

  useImperativeHandle<T, any>(ref, () => ({ toggle }));

  const closeModal = useCallback(() => toggle(false), []);

  return { visible, closeModal, toggle };
};

export default useModalToggle;

// 在demoModal中使用
export default React.memo(forwardRef<ModalRef, Props>((props, ref) => {
  const { onConfirm } = props!;
  // 暴露出一个状态和一个关闭事件，其实还有第三个参数，不过这个例子没有用到。并减少组件里包的import
  const { visible, closeModal } = useModalToggle(ref);

  const confirmHandle = useCallback(() => {
    onConfirm('传递参数给父组件');
  }, [onConfirm]);

  return (
    <Modal
      visible={visible}
      title="弹框标题"
      onCancel={closeModal}
      footer={[
        <Button type="primary" key="submit" onClick={confirmHandle}>确定</Button>,
        <Button key="reset" onClick={closeModal}>取消</Button>
      ]}
    >
      这是弹框内容
    </Modal>
  );
}));
```

```js
// useModalRef.ts
import React, { useRef } from 'react';
// 这里的ModalRef应该作为一个通用的类型，这里只是作为例子，所以直接写了
interface ModalRef {
  toggle: (value?: boolean) => void
}
type Toggle = ((value?: boolean) => void);
type UseModalRef = () => [React.RefObject<ModalRef>, Toggle];

const useModalRef: UseModalRef = () => {
  const ref = useRef<ModalRef>(null);

  const refToggle = (value?: boolean) => {
    if (ref.current) {
      ref.current.toggle(value);
    } else {
      console.error('该组件Ref还未挂载完成');
    }
  };

  return [ref, refToggle];
};

export default useModalRef;

// 在App.ts里使用
export default () => {
  // 暴露出Ref和toggle方法，简化之前的写法，并减少组件里包的import
  const [demoModalRef, demoModalToggle] = useModalRef();

  const modalConfirm = useCallback((val: string) => {
    console.log(val);
  }, []);

  return (
    <>
      <div onClick={() => demoModalToggle(true)}>点击打开弹框</div>
      <DemoModal
        ref={demoModalRef}
        onConfirm={modalConfirm}
      />
    </>
  );
};
```

这里的重点主要还是从父组件单向的向子组件传递参数和方法的思想中跳脱出去，有时候让子组件暴露方法给父组件去使用能够更好的解决你业务上的问题。

当然这里还可以进一步再做处理，比如将需要传入子组件的参数和方法都通过`ref`方法传入，不过这样虽然子组件只需要绑定一个`ref`属性，但是因为需要处理不同`Modal`的参数差异，耦合性会变得过大，不太符合封装的设计模式，所以不是很推荐。不过如果有一些相对独立的组件，可以在`ts`完备的情况下尝试使用一下。

## 总结
本文介绍了一些这段时间对于`useRef`的学习以及实际的使用，如果有描述错误的地方，可以在下面评论指出，我会及时回应并做修改。

## 感谢
如果你觉得这篇文章对你有所帮助，请点个赞，谢谢各位。