## 前言
介绍一些我在`js`编程中常用的一些设计模式，**本文没有理论的设计模式的知识，每个模式都会从实际的例子出发，说明为什么要使用相应的设计模式？怎么去使用？**

大家也不要觉得设计模式很难，很高级，之所以觉得“难”，只是因为纯理论知识的枯燥难懂，我会从实际例子出发，用非常接地气的方式，给大家列举一些我们平时常用，好用的一些设计模式的具体实践。
## 设计模式简介
简单介绍一下设计模式，指导理论一共有5个基本原则
- 单一功能原则
- 开放封闭原则
- 里式替换原则
- 接口隔离原则
- 依赖反转原则

23个经典的模式
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f077a14fec26480e8bda2639e33b1d4e~tplv-k3u1fbpfcp-watermark.image)
这些内容看过一遍就行，不需要深入去了解。对于基本原则，在`js`的编程设计中，了解“单一功能“和“开放封闭”基本就够用。对于模式上，很多的模式其实我也根本没有使用过，因为设计模式的产生初衷，是为了补充 Java 这样的静态语言的不足。许多"经典"设计模式，在编程语言的演化中，早已成为语言机制的一部分。比如，export 内建了对单例模式的支持、将内容用 function 包装一层就是工厂模式、yield 也实现了迭代器模式等等。
## 为什么要使用设计模式

设计模式的核心思想只有一个，那就是**封装变化**。借用修言大佬的话
>实际开发中，不发生变化的代码可以说是不存在的。我们能做的只有将这个变化造成的影响最小化 —— 将变与不变分离，确保变化的部分灵活、不变的部分稳定。

这就是平时我们常说的“健壮”的代码，而设计模式就是帮助我们实现这个目的的工具。
## 简单工厂模式
不说理论，直接上例子。

>在王者荣耀里，根据每个人的星级数量，都会有一个排位的等级，现在比如有三个等级，黄金，钻石，王者。它们三个有一点区别，黄金段位是全英雄匹配，钻石和王者是BP模式的匹配。王者段位可以进行巅峰赛，但是其他两个不行。现在有个需求，让你通过段位来返回一个相应的实例，而且需要符合这些区别？

这对于我们来说，也太轻松了吧。噼里啪啦几几分钟，代码就写好了。
```js
class 黄金 {
  constructor() {
    this.level = '黄金'
    this.ifBP = false
    this.canJoinPeaked = false
  }
}
class 钻石 {
  constructor() {
    this.level = '钻石'
    this.ifBP = true
    this.canJoinPeaked = false
  }
}
class 王者 {
  constructor() {
    this.level = '王者'
    this.ifBP = true
    this.canJoinPeaked = true
  }
}
function Factory(level) {
  switch(level) {
    case '黄金':
      return new 黄金()
      break
    case '钻石':
      return new 钻石()
      break
    case '王者':
      return new 王者()
      break
  }
}	
```
>后面王者更新了，如果新增了10个新的段位，你要怎么改这个代码？还是一个个手动添加吗？

现在让我们来改造一下
```js
class 段位通用类 {
  constructor(level, ifBP, canJoinPeaked) {
    this.level = level
    this.ifBP = ifBP
    this.canJoinPeaked = canJoinPeaked
  }
}
function Factory(level) {
	let ifBP, canJoinPeaked
  switch(level) {
    case '黄金':
      ifBP = false
      canJoinPeaked = false
      break
    case '钻石':
      ifBP = true
      canJoinPeaked = false
      break
    case '王者':
      ifBP = true
      canJoinPeaked = true
      break
  }
  return new 段位通用类(level, ifBP, canJoinPeaked)
}	
```
这个就是简单工厂模式的具体应用，将**创建对象的过程封装**，我们不需要去关心具体的内容，只要传入参数，拿到工厂给我们的对象即可。
## 策略模式
> 王者荣耀里，我们如果进行排位赛，会根据你的段位去匹配一起游戏的玩家，现在有个需求，要求写一个排位匹配函数，根据玩家当前的段位等级，来执行不同段位的排位匹配功能？

这对于习惯了if-else的我们来说，也是如此简单。
```js
class 王者账号 {
  constructor() {}
  排位匹配(level) {
    if (level === '黄金') {
      console.log('执行黄金段位的匹配')
      // 这里只是举个例子，平时开发，这里可能会有很长一段的复杂代码逻辑
    }
    if (level === '钻石') {
      console.log('执行钻石段位的匹配')
    }
    if (level === '王者') {
      console.log('执行王者段位的匹配')
    }
  }
}
王者账号.排位匹配('黄金')
```
代码写完了，功能实现了，运行起来的确没问题。但是其实这里存在多个隐患。
- 没有遵循单一功能原则，这里在一个函数里处理了多种情况的逻辑，万一其中有一个出了bug，后续的逻辑就都无法运行了。而且功能都放在一起，功能的抽离复用变得很困难。
- 没有遵循开放封闭原则（只新建，不修改），如果后续又多了一个段位，只能继续通过if去判断，导致每次新增都要对这个排位匹配函数进行测试回归，增加工作量。

现在我们来对其进行改造，首先遵循单一功能原则，把每一项的功能逻辑抽离出来。
```js
function 黄金匹配() {
  console.log('执行黄金段位的匹配')
}
function 钻石匹配() {
  console.log('执行钻石段位的匹配')
}
function 王者匹配() {
  console.log('执行王者段位的匹配')
}
class 王者账号 {
  constructor() {}
  排位匹配(level) {
    if (level === '黄金') {
      黄金匹配()
    }
    if (level === '钻石') {
      钻石匹配()
    }
    if (level === '王者') {
      王者匹配()
    }
  }
}
王者账号.排位匹配('黄金')
```
接下来，我们来遵循开放封闭原则（只新建，不修改），**封装变化**
```js
const 匹配逻辑 = {
  黄金() {
    console.log('执行黄金段位的匹配')
  },
  钻石() {
    console.log('执行钻石段位的匹配')
  },
  王者() {
    console.log('执行王者段位的匹配')
  },
}
class 王者账号 {
  constructor() {}
  排位匹配(level) {
    匹配逻辑[level]()
  }
}
王者账号.排位匹配('黄金')
```
改动之后，后续不管是新增还是删除，我们都不需要去修改`排位匹配`这个函数，只用对`匹配逻辑`进行修改就好。

**策略模式的核心就是把变化算法提取封装好，并是让其可替换。适合表单验证、或者存在大量 if-else 的场景使用。**
## 状态模式
状态模式跟策略模式其实没啥本质上差别，但是多了一个状态的概念，我们还是刚上一个`排位匹配`的代码来示例。
> 王者里有个机制，信誉分，信誉分过低系统会禁止玩家排位功能。我这边稍作修改来当做例子，王者账号这个类里有一个信誉分的参数，信誉分达到80分，黄金段位可以排位，信誉分达到90分钻石段位才可以排位，信誉分达到100分，王者段位才可以排位。现在要求实现这个逻辑？
```js
const 匹配逻辑 = {
  黄金() {
    console.log('执行黄金段位的匹配')
  },
  钻石() {
    console.log('执行钻石段位的匹配')
  },
  王者() {
    console.log('执行王者段位的匹配')
  },
}
class 王者账号 {
  constructor() {
    this.creditPoints = 80
    //80黄金，90钻石，100王者
  }
  排位匹配(level) {
    匹配逻辑[level]()
  }
}
```
通过上面的练习，现在大家应该想到，要把这单个的功能在到`匹配逻辑`里的各项里去，这样后续如果有段位的新增和删除，或者信誉分逻辑的修改，我们都不需要去修改`排位匹配`这个函数，可以减少测试的工作量。

但是这个信誉分的状态怎么拿到呢？现在我们来使用状态模式的思想来改造一下。
```js
class 王者账号 {
  constructor() {
    this.creditPoints = 80
    //80黄金，90钻石，100王者
  }
  匹配逻辑 = {
    that: this,
    黄金() {
      if (this.that.creditPoints >= 80) {
        console.log('执行黄金段位的匹配')
      }
    },
    钻石() {
      if (this.that.creditPoints >= 90) {
        console.log('执行钻石段位的匹配')
      }
    },
    王者() {
      if (this.that.creditPoints >= 100) {
        console.log('执行王者段位的匹配')
      }
    }
  }
  排位匹配(level) {
    匹配逻辑[level]()
  }
}
```
是不是很简单？

**状态模式主要解决的是当控制一个对象状态的条件表达式过于复杂时的情况。把状态的判断逻辑转移到表示不同状态的一系列类中，可以把复杂的判断逻辑简化。**

## 单例模式
> 要求实现一个全局唯一的Modal弹框

这是一道非常经典的单例模式的例子，也是比较常见的面试题。直接上答案了。
```js
const SingleModal = (function() {
  let modal
  // 利用闭包实现单例
  return function() {
    if(!modal) {
        modal = document.createElement('div')
        modal.innerHTML = '全局唯一的Modal'
        modal.style.display = 'none'
        document.body.appendChild(modal)
    }
    return modal
  }
})()
// 创建和显示
const modal = SingleModal()
modal.style.display = 'block'

// 隐藏
const modal = SingleModal()
modal.style.display = 'none'
```
后续每次调`SingleModal()`返回的都是第一次运行时创建的那个Modal弹框。也可以使用类的方式实现单例。
```js
class SingleModal{
  // 这里是定义了一个静态方法，也可以写在类的构造函数里。大家可以自己试着写一下
  static createModal() {
      if (!SingleModal.instance) {
        let modal
        modal = document.createElement('div')
        modal.innerHTML = '全局唯一的Modal'
        modal.style.display = 'none'
        document.body.appendChild(modal)
        SingleModal.instance = modal
      }
      return SingleModal.instance
  }
}
const modal1 = SingleModal.createModal()
const modal2 = SingleModal.createModal()

modal1 === modal2 // true
```

**单例模式的目的就是保障不管多少次的调用，返回的都是同一个实例。**

像`vuex`就是典型的单例实现，所有子组件访问到的store其实都是根组件的那个store实例，修改的都是同一个由`vuex`创建出来的`vue`实例。

## 装饰器模式
>王者荣耀里，基本每个英雄都有好几套皮肤，酷炫的皮肤带来了更佳的游戏体验。拿我最喜欢的英雄李白为了例子，我现在假设出了一个神级皮肤，换上这套皮肤之后，李白会再多出一个技能，这个技能的效果就是“嘲讽”，而且没有cd，无限的嘲讽攻击，让对手失去理智。要求实现这个皮肤的效果？

先来一个李白实例,原本有三个技能。
```js
class 李白 {
  技能1() {
    console.log('将进酒')
  }
  技能2() {
    console.log('神来之笔')
  }
  技能3() {
    console.log('青莲剑歌')
  }
}
```
现在要求根据是否使用了这个皮肤来判断，是否要添加“嘲讽”这个技能。怎么写？

很轻松嘛，根据皮肤状态来判断一下就ok嘛。
```js
class 李白 {
  constructor(skin) {
    if (skin === '神级皮肤') {
      this.嘲讽 = () => {
        console.log('释放嘲讽')
      }
    }
  }
  技能1() {
    console.log('将进酒')
  }
  技能2() {
    console.log('神来之笔')
  }
  技能3() {
    console.log('青莲剑歌')
  }
}
```
首先，这个实现，违背了开放封闭原则，我们希望能够遵守“只新增，不修改”的原则。其次，“嘲讽”这种作为普适性很强的行为很可能会被加到其他的英雄上面去，比如后续需求变更了，所有的英雄都会出一个神级皮肤，都需要有一个嘲讽技能怎么办？所有的英雄一个个去加吗？

这时候，我们可以使用装饰器的思想去改造。
```js
class 李白 {
  技能1() {
    console.log('将进酒')
  }
  技能2() {
    console.log('神来之笔')
  }
  技能3() {
    console.log('青莲剑歌')
  }
}
class 嘲讽技能装饰器 {
  constructor(hero) {
    this.hero = hero
  }
  技能1() {
    this.hero.技能1()
  }
  技能2() {
    this.hero.技能2()
  }
  技能3() {
    this.hero.技能3()
  }
  嘲讽() {
    console.log('释放嘲讽')
  }
}
let hero = new 李白()
if (skin === '神级皮肤') {
  hero = new 嘲讽技能装饰器(hero)
  hero.嘲讽()
}
```
这样，我们没有对李白这个实例进行任何的修改，只是新增了一个装饰器，而且这个装饰品还可以复用于所有其他英雄的实例上。

**装饰器的核心思想就是不对原先的功能有任何的影响，只使其具备新的能力。**

es7中，js可以通过@语法糖对类或者类中的函数方法添加装饰器。这块内容大家有兴趣的话自己去了解一下，篇幅限制，这里就不细讲了。给大家推荐一个优秀的第三方装饰器库 [core-decorators](https://github.com/jayphelps/core-decorators)。

装饰器的应用很广泛，再讲一些其他例子。
```js
// 对于Math.abs来说，add也算一个装饰器
const add = (a, b, abs) => {
    return abs(x) + abs(y);
}
const num = add(1, -1, Math.abs);
```
```js
// react里很常见的高阶组件，也是装饰器的一个应用
const withDoSomthing = (component) => {
  const NewComponent = (props) =>{
    return <component {...props} />
  }
  return NewComponent
}
```
## 适配器模式
适配器主要是为了解决兼容性的问题，帮助我们抹平差异。

举个例子，我用的是苹果手机，充电口是Lightning接口。今天我一不小心，把我的苹果充电线弄断了。手机快没电了，可我这局王者才开始，这一局是进阶赛，赢了就上王者了。可我家里只有一根安卓的type-c充电线，还有一根usb的充电线。我看着1%的电量感慨到，如果能有一个转换头，能把type-c以及usb的接口转换成苹果的Lightning接口那该有多好。

这个转换头就是适配器。这边再举两个实际的例子给大家参考。

**jquery的each遍历**

大家对`forEach`应该特别熟悉，我们在遍历数组的时候经常会用到，比如
```js
let arr = ['a', 'b', 'c']
arr.forEach(item => {
  console.log(item)
})
```
但是如果我们换一个对象
```js
const divList = document.getElementsByTagName('div')
for (let i = 0;i < divList.length;i ++) {
  console.log(divList[i])
}
// 正常
document.getElementsByTagName('div').forEach(item => {
  console.log(item)
})
// Uncaught TypeError: document.getElementsByTagName(...).forEach is not a function
```
我们会发现,`for`方法可以正常打印所有的div标签。但是`forEach`方法会报错，为什么呢？

因为这里的`divList`是一个类数组对象，它本质上是一个对象，只是它的`key`是0,1,2这种格式，而且存在`length`属性。既然它不是数组，我们当然不能用`forEach`来对她进行遍历。

但如果我们使用`jquery`的`each`方法。
```js
const arr = ['a', 'b', 'c']
const divList = document.getElementsByTagName('div')
$.each(arr, function (index, item) {
  console.log(item)
})
// 正常遍历
$.each(divList, function (index, item) {
  console.log(item)
})
// 正常遍历
```
我们发现对于这两种类型，都可以进行遍历，这是因为`jquery`的`each`内部已经帮我们抹平了差异，我可以使用同样的方法来读取不同类型的列表数据。这就是适配器的典型表现。

**axios**

axios的不同配置方式也是适配器的一种表现形式。
```js
 axios({
   url: '/post',
   method: 'post',
   data: {
     msg: 'hello'
   }
 })
 axios('/post', {
   method: 'post',
   data: {
     msg: 'hello'
   }
 })
 axios.request({
   url: '/post',
   method: 'post',
   data: {
     msg: 'hello'
   }
 })
 axios.post('/post', { msg: 'hello' })
```
上面4种配置方式，都可以实现相同的接口调用，不愧是`axios`啊。

## 代理模式
代理模式在平时的开发中也是应用非常广泛，而且代理模式的理念能够带来非常直接的性能提升，非常实用。

**事件代理**

利用点击事件的冒泡机制实现的事件代理，这个太基础了，不细讲了，略过。

**缓存代理**

把一些计算频繁的模块内容存下来，等到下次用到了，直接读取，不再二次计算，看几个具体例子吧。
```js
// 最常见，最简单的缓存代理
for (let i = 0; i < document.getElementsByTagName('div').length;i ++) {
  console.log(document.getElementsByTagName('div')[i])
}
// 使用缓存代理
const divList = document.getElementsByTagName('div')
for (let i = 0; i < divList.length;i ++) {
  console.log(divList[i])
}

// 对一些需要遍历对象深层次数据也是同理
const obj = { child: { child: { child: [1,2,3,4,5] } } }
const childList = obj.child.child.child
for (let i = 0; i < childList.length;i ++) {
  console.log(childList[i])
}
```
进阶版的缓存代理，取自[修言大佬](https://juejin.im/user/2400989094885495)的JavaScript 设计模式核⼼原理与应⽤实践小册
```js
// 计算所有参数之和
const addAll = function() {
    let result = 0
    const len = arguments.length
    for(let i = 0; i < len; i++) {
        result += arguments[i]
    }
    return result
}
// 为求和方法创建代理
const proxyAddAll = (function(){
    // 求和结果的缓存池
    const resultCache = {}
    return function() {
        // 将入参转化为一个唯一的入参字符串
        const args = Array.prototype.join.call(arguments, ',')
        
        // 检查本次入参是否有对应的计算结果
        if(args in resultCache) {
            // 如果有，则返回缓存池里现成的结果
            return resultCache[args]
        }
        return resultCache[args] = addAll(...arguments)
    }
})()
```
缓存代理可以减少二次计算，提高性能，真是太实用了。

`vue`在生成子组件的时候，就是使用了缓存代理，在第一次生成子组件之后，后面如果需要再次生成该子组件，`vue`会从缓存当中返回子组件实例，避免了组件生成逻辑的重新计算。

**拦截代理**

拦截代理其实在`es6`之前其实没啥很特别的表现形式，具体的形式，其实就只是一些判断而已。
```js
function 吃饭() {
  console.log('吃饭')
}
// 这个方法其实就是拦截代理
function 我要不要吃饭(status) {
  if (status === '我饿了') {
    吃饭()
  }
}
function 午饭时间到了(status) {
  我要不要吃饭()
}
```
`es6`之后，我们有一个新的拦截器的方法，`Proxy`。

这边举一个最常见的`set`和`get`的例子。
```js
let myMessage = { name: "zouwowo", age: 27, sex: '男' }
// 添加Proxy拦截
let message = new Proxy(myMessage, {
  get(target, key) {
    if (key === 'age') {
      // 只要获取我的age，永远都是18岁
      return 18
    } else {
      return target[key]
    }
  },
  set(target, key, value) {
    if (key === 'sex') {
      // 我是男的，怎么改，都改不了性别
      target[key] = '男'
    } else {
      target[key] = value
    }
  }
});
console.log(message.age) // 不管myMessage变量里的age是多少，永远返回的是18岁
message.sex = '女' // myMessage里的sex不会被修改
```

`Proxy`有10多种监听拦截的方法，有兴趣的同学可以去了解学习一下。`vue3`的数据监听也从`Object.defineProperty`方法改到了`Proxy`，解决了之前新增的深度数据，部分数组修改方法无法监听的问题，足以见其的强大。

## 观察者模式
> 小明昨天玩王者荣誉，被对面有神级皮肤的李白疯狂嘲讽，一整场下来，被李白杀了10多次，“一群菜鸡队友，不然肯定吊打这个XX李白。”小明气不过，加了李白好友，约定组个队再打一局。小明找了自己好友里段位最高的四个人，小王、小者、小荣、小耀。五个人一起拉了一个微信群，小明说：“大家稍等一会儿，等要开打了，叫你们”。四个人各自忙自己的事情去了，然后等到晚上9点，小明在群里一吼：“兄弟们上号！”。四人收到了消息，各自上号。最终小明依然经历了一次边被嘲讽，边被虐杀的游戏体验。

上面这个例子，就是一个典型的观察者模式。

在上述的过程中，`发布者`只有一个——小明，但是`观察者`有多个，小王、小者、小荣和小耀。`发布者`发布事件，所有`观察者`都能通过微信群观察到`发布者`的指令，然后执行各自的任务（各自上号）。

我们来整理一下`发布者`和`观察者`各自都需要实现什么功能。

`发布者`需要两个功能
1. 创建微信群（添加观察者）
2. 通知上号（发布事件）

`观察者`需要两个功能
1. 等待群主发布通知上号（接受通知）
2. 各自上号（执行各自的任务）

现在我们来实现这个最简单的观察者模式
```js
class 发布者 {
  constructor() {
    this.observers = []
  }
  // 添加观察者
  addObserver(observer) {
    this.observers.push(observer)
  }
  // 发布事件，通知所有的观察者
  notify() {
    this.observerList.forEach(observer => observer.update())
  }
}
class 观察者 {
  constructor(work) {
    this.work = work
  }
  update() {
      console.log(this.work)
  }
}
const 小明 = new 发布者()
const 小王 = new 观察者('辅助')
const 小者 = new 观察者('打野')
const 小荣 = new 观察者('中单')
const 小耀 = new 观察者('上单')
// 小明创建微信群，拉人
小明.addObserver(小王)
小明.addObserver(小者)
小明.addObserver(小荣)
小明.addObserver(小耀)
// 小明通知群里的所有人上号，群里的人，各自完成自己的任务
小明.notify()
```

**观察者模式的核心思想就是这种一对多的关系，当发布者发布事件，所有的观察者都会自动完成更新。**

vue的响应式依赖实现的核心，就是Dep类，Watch类和Object.defineProperty这三者实现的观察者模式。

## 发布订阅模式
发布订阅模式实现的也是这种事件的发布和订阅功能。这个比较好理解，直接看代码吧。
```js
class EventBus {
  constructor() {
    // 存放所有的事件
    this.events = {}
  }
  // 发布事件
  subscribe(event, fn) {
    if ( !this.events[event] ) {
        this.events[event] = []
    }
    // 将事件函数放入该事件名的数组里
    this.events[event].push(fn)
  }
  // 订阅事件
  publish(event, ...args) {
    if (this.events[event] ) {
      // 调用该事件名下的所有事件
      this.events[event].forEach( fn => fn(...args) )
    }
  }
  // 删除事件名下某个事件
  unsubscribe(event, fn) {
    if (this.events[event]) {
      const targetIndex = this.events[event].findIndex(item => item === fn) 
      if (targetIndex !== -1) {
        this.events[event].splice(targetIndex, 1)
      }
      // 该事件名下无事件时直接删除该订阅事件
      if (this.events[event].length === 0) {
        delete this.events[event]
      }
    }
  }
  // 删除某个事件名下的所有事件
  unsubscribeAll(event) {
    if (this.events[event]) {
      delete this.events[event]
    }
  }
}
```
具体使用
```js
const event = new EventBus()
event.subscribe('aaa', ()=> console.log('我订阅了aaa事件'))
event.subscribe('aaa', ()=> console.log('我又订阅了aaa事件'))
event.publish('aaa')
// 打印: 我订阅了aaa事件
// 打印: 我又订阅了aaa事件
```

相比较`可观察者模式`，`发布订阅模式`除了发布者和订阅者之外，多了一个事件中心。发布者和订阅者之间没有任何的关联，两者只能通过事件中心去进行通信。

vue内部也实现了一个`发布订阅模式`，`$on`,`$emit`就是对应的发布者和订阅者。
## 写在最后
各个设计模式并不是完成分离的，它们都是相辅相成，可以互相套用的。比如策略模式，只要适合，可以用在其它的设计模式里。

这篇文章不是让大家强行套用设计模式，我们需要记住的不是这一个个设计模式的名字，也不是为了在看一些别人写的优秀代码的时候一定要分辨出这是使用了哪个设计模式。设计模式只是手段，帮助我们写出"优雅"代码的手段，我们只需要记住这些设计模式的核心思想。君子善假于物，但是不能被“物”所束缚，而且千万要避免过度设计。

设计是一个循序渐进的过程，是从不断的试错当中来的，前期再完美的设计并不能满足中后期大量的需求变更，产品的一个需求，可能就能把你之前完美的设计打破，所以不要指望一下把所有细节都设计出来，边写边重构才是我们项目开发中的一个好习惯。
## 参考文章 
修言大佬  [javaScript设计模式核心原理与应用实践](https://juejin.im/book/6844733790204461070)
## 感谢
感谢大家的阅读，如果觉得不错的话，帮忙点个赞，给咱一点支持，谢谢！
