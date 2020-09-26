我最近在面试，也有碰到不少自己不太清楚的题目，这是其中一道，难度其实并不是特别高，主要是分享一下我解决这个问题的具体流程，供一些初中级的前端同学有所参考。

## 1、问题
```html
<p>{{a.b.c}}</p>
<input type="text" v-model="a.b.c">
<button @click="resetData">重置数据</button>
```
```js
var vm = new Vue({
  el:"#app-vue",
  data:{
    a: {
      b: {
        c: 1
      }
    }
  },
  methods: {
    resetData() {
      //假设这个data是接口返回出来的值
      let data = {
        b: {
          c: 2
        }
      }
      this.a = data
    }
  }
})
```
这个问题是这样的，**给a对象赋值之后，为什么里面的c都还是响应式的**？

这种情况其实在平时的开发过程中应该非常常见，从后端接口里把值取出来，然后给data赋值。
<br/>
从源码层面上来看，vue里面的响应式是通过`Object.defineProperty()`方法遍历了data中的数据，把每一个变量的set和get方法劫持，添加了dep.depend和dep.notify相关的操作。
<br/>
但是对于这道题，我一下子不知道如何回答。对c的set和get的劫持是在b参数上的，但是b已经被重新赋值了，正常来说其实应该不会有响应式属性了。

## 2、分析和解决过程
### 2.1 搜索
对于这些问题，第一印象肯定是去网上搜索一下，看看有没有相关的帖子或者博客，而且当时我还赶着后面的面试，就希望能够直接找到答案，但是网上找不到相关的内容。
### 2.2 defineProperty
晚上到家之后，开始试着自己去解决问题。解决之前，先对问题进行了分析，首先我猜测是不是`defineProperty`的原因，定义之后，就会一直保存住，后续的数据更新，只要是符合定义的命名的就都会触发相应的方法。在浏览器控制台上测试了一下，并不是它的原因，重新赋值就不会有劫持的set和get了。
### 2.3 debugger
既然不是这个原因，我就先通过debugger调试，看一下b这个参数在set前后是否有发生一些属性的变化。<br/>
set前
![](https://imgkr.cn-bj.ufileos.com/1d564c36-eeed-4fe8-b0fc-e0ab2e704dca.png)
set后
![](https://imgkr.cn-bj.ufileos.com/b5bce5ad-ae77-48bb-a6f4-a081661a040e.png)
在这里，我们发现b这个参数中的dep的id还有subs里的数组项是不一致的，所以可以确认，是vue在变量的set过程中有做了一些处理
### 2.3 vue源码
看[Vue源码](https://github.com/vuejs/vue)之前，我们都分析一下，这个功能是属于那一块的。很明显，这是响应式初始化的功能，所以需要到`core/observer/index.js`文件下去看，在里面，我们会找到一个`defineReactive`这个就是我们初始化响应式的方法。然后看里面定义的set方法。（只放了核心功能的代码，完整的请看源码）
```javascript
 set: function reactiveSetter (newVal) {
      val = newVal
      //本问题的关键代码
      childOb = !shallow && observe(newVal)
      dep.notify()
    }
```
分析代码，很容易看到肯定是`observe`中做了一些处理，`shallow`这个值是用来判断当前赋值的变量是否是vue的根data下的变量，然后我们来查看`observe`这个方法（只放了核心功能的代码，完整的请看源码）

```js
export function observe (value: any, asRootData: ?boolean): Observer | void {
  //判断set传入的value是否是一个对象或者是一个VNode
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  //判断set传入的value是否是一个已经初始化响应式依赖的属性
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if {
  //调用Observer方法
    ob = new Observer(value)
  }
  return ob
}
```
我们继续看`Observer`这个方法，这里是响应式初始化的内容（只放了核心功能的代码，完整的请看源码）
```js
export class Observer {
  constructor (value: any) {
    //判断value是否是一个Array
    if (Array.isArray(value)) {
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }
  //遍历对象来添加响应式
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }
  //遍历数组来添加响应式
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

## 3、总结
通过上面的源码（vue2.6）的分析，我们现在可以知道，问题中的c依然还有响应式的原因是vue在data进行set赋值的时候，还会通过observe方法去判断这个新值是否是对象，而且是否已经有响应式初始化。如果没有，重新对其进行响应式初始化。

## 4、后感
其实对vue源码的内容，我看过很多，但是很多都是关于那种大的功能上的实现。通过这个问题，我发现自己其实对于这些细节方面的实现，还是会有很多的遗漏和疏忽。虽然马上vue3的正式版出来，但是对于vue3相关的ui框架稳定之前的较长的一段时间内，大部分的公司应该都不会轻易的升级vue版本。所以大家应该还需要抽出一点时间来对vue2的源码层面的东西继续查漏补缺。

## 5、感谢
首先感谢您的浏览，希望您能够动动小手给咱点个赞，给我这样一个掘金新人一点支持，谢谢大家。