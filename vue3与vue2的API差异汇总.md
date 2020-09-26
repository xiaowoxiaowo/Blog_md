## 一.概要

上周，`vue3 one piece`正式版发布了（话说尤大这个用动漫名定义版本号的习俗什么时候轮到火影啊啊啊），在给尤大疯狂打call的同时，也是咱们开始爆肝学习的时候了。

我对比了`vue2`和`vue3`，在这里列举出使用方法，功能或者已经废弃的一些`api`。供一些`vue2`的同学快速的了解和上手`vue3`。在这篇文章里，**只列出一些使用上有所差异的内容，不涉及到源码方面。**

除了安装以及vue3一些全新的方法外，API方面我会按照[vue2的官方api文档](https://cn.vuejs.org/v2/api/)的目录顺序来列举。
- 安装
- vue3新内容
- 应用配置（原全局配置和全局API）
- 全局API
- 选项/数据
- 选项/DOM
- 选项/生命周期钩子
- 选项/资源
- 选项/组合
- 选项/其它
- 实例property
- 实例方法/数据
- 实例方法/事件
- 实例方法/生命周期
- 指令
- 特殊attribute
- 内置的组件

本人技术有限，这里有些api可能我也不常用到或者没怎么使用过，所以如果有所错误或者遗漏，请在下面评论指出，我会及时进行修改。

这里是我个人的理解和记录，现在[vue3官方文档](https://v3.cn.vuejs.org/)已出，大家也可以去官网查看vue2的迁移内容。

## 二.安装

项目的安装现在有三种方式。
- 通过script引入
```js
<script src="https://unpkg.com/vue@next"></script>
```
- 脚手架vue-cli
```js
// vue-cli在4.5.0的版本才开始支持创建vue3项目
npm install -g @vue/cli
vue create hello-vue3
```
- 脚手架vite
```js
npm init vite-app hello-vue3
```

vite是尤大新的开源工具，借用尤大的原话
>一个基于浏览器原生 ES imports 的开发服务器。利用浏览器去解析 imports，在服务器端按需编译返回，完全跳过了打包这个概念，服务器随起随用。同时不仅有 Vue 文件支持，还搞定了热更新，而且热更新的速度不会随着模块增多而变慢。针对生产环境则可以把同一份代码用 rollup 打包。虽然现在还比较粗糙，但这个方向我觉得是有潜力的，做得好可以彻底解决改一行代码等半天热更新的问题。

总的来说，就是js的引入使用了浏览器自身的es imports方法。开发模式下不需要打包，只需要编译修改过的文件即可。生产环境下使用rollup打包。

这是一个新的方向，虽然目前还很稚嫩，而且对于ES imports浏览器的兼容还是很有问题，不过再过几年，这可能也会变成一个主流的工具。


## 三.vue3新内容

### 1.Composition API
将数据、方法、computed、生命周期函数,集中写在一个地方。
#### 1.1 setup()
`setup()`作为在组件内使用`Composition API`的入口点。执行时机是在`beforeCreate`和`created`之间,不能使用this获取组件的其他变量，而且不能是异步。`template`中使用到的值，都必须是`setup`返回的对象（除了props）。

setup有两个参数,`props`,`context`。

```js
<script lang="js">
import {toRefs} from 'vue'
export default {
    name: 'demo',
    props:{
      name: String,
    },
    setup(props, context){
      // 这里需要使用toRefs来进行解构
      // 这里的props与vue2基本一致,当然这里的name也可以直接在template中使用
      const { name }=toRefs(props);
      console.log(name.value);
      // 只能获取到这三个属性,也是不能使用ES6的解构
      // 属性，同vue2的$attrs
      console.log(context.attrs);
      // 插槽
      console.log(context.slots);
      // 事件，同vue2的$emit
      console.log(context.emit);
  }
}
</script>
```

#### 1.2 reactive

`vue3`将响应式的功能和vue代码解耦，单独作为一个js库来调用。（在其他任何js运行的框架当中，都可以引入和使用`vue3`的这个响应式功能）。
```js
<template>
  <div>{{ state.text }}</div>
  <button @click="test">测试一下？</button>
</template>
<script>
import { reactive } from 'vue';
export default {
  setup() {
    // 通过reactive声明一个可响应式的对象
    const state = reactive({ text: "我帅吗" });
    // 通过reactive声明一个可响应式的数组
    const state1 = reactive([ text: "我帅吗" ]);
    // 声明一个修改方法
    const test = () => state.text = '你很丑';
    // 返回state和方法
    return { state,test };
  }
};
</script>
```
这是一个跟`vue2`比较大的区别，使用上面的方法，才能实现数据的响应式。然后这里的对象可以随意赋值，不会出现`vue2`中深层未定义的对象值赋值不能响应式的问题。因为现在vue的底层是用`proxy`实现的数据监听。

**注意：这里的元素只能是对象或者数组，基本数据类型需要使用ref**

#### 1.3 ref
声明响应式的基本数据类型
```js
import { ref } from 'vue';
export default {
  setup(){
    const a = ref(1);
    const b = ref('1');
    const c = ref(true);
    // 声明一个修改方法
    const test = () => {
      // 注意：在setup里面修改ref的值是需要通过.value的，template做了解套，所以不需要。
      // ref 作为 reactive的对象的值时不需要.value
      // ref 作为 reactive的数组的值时需要.value
      a.value = 2;
      b.value = '1';
      c.value = false;
    };
    return { a, b, c, test };
  }
}
```
获取DOM元素
```js
<template>
  <div id="test" ref="testDom"></div>
</template>

<script>
import { ref, onMounted } from 'vue';
export default {
  setup(){
    const testDom = ref(null);
    //使用onMounted钩子,类似在vue2的mounted生命周期中操作元素
    onMounted(() => {
      // 记得要加.value
      console.log(testDom.value);
    })
    return { testDom };
  }
}
</script>
```
#### 1.4 readonly
可以传入一个对象或ref，返回一个只读的对象，深层次的只读
```js
const state = reactive({ name: 0 })
const readOnlyState = readonly(state)
// 可以修改
state.name = '帅';
// 无法修改并警告
readOnlyState.name = '帅';
```
### 2 reactive类工具API
#### 2.1 isProxy， isReactive，isReadonly
isProxy是检查一个对象是否是由 reactive 或者 readonly 方法创建的代理。后面两个是单独的检测。
注意：只要检测的对象里有reactive，就算被readonly处理过了，isReactive也为true。
#### 2.2 toRaw
把被代理或者响应式的对象转换成普通对象
```js
const state = {}
const stateReactive = reactive(state)
console.log(toRaw(stateReactive) === state) // true
```
#### 2.3 markRaw
禁止这个对象成为响应式

```js
const state = markRaw({ name: '帅' })
console.log(isReactive(reactive(state))) // false
```
**注意，markRaw只作用于根元素，如果使用子元素去赋值，还是可以转成响应式代理**
```js
const state = markRaw({
  data: { id: 123 },
})
const newState = reactive({
  data: state.data,
})
console.log(isReactive(newState.data)) // true
```
#### 2.4 shallowReactive, shallowReadonly
跟上面`reactive`,`readonly`的作用一样，区别是这两个方法是浅层的监听。
只会对第一层响应式，深层次的数据不会响应
```js
const state = shallowReactive({ name: '123', test: { id: 1 } })
const test = () => {
  // 深层次数据的变化不会触发响应式
  state.test.id = 2;
};
```
### 3 ref类工具API
#### 3.1 isRef
检查一个值是否为一个`ref`对象
#### 3.2 toRef
用来作为一个`reactive`对象的属性,保持响应式
```js
const state = reactive({
  a: 0
})
// 作为一个reactive对象的属性
const aRef = toRef(state, 'a')
// 在这里修改读取Ref要使用.value
aRef.value++
console.log(state.a) // 1

state.a++
console.log(aRef.value) // 2
```
#### 3.3 toRefs
将响应式对象转换为普通对象(类似解构)，但是还可以保留响应式。可以用来对一些响应式变量多次复用。
```js
function useFeatureX() {
  const state = reactive({
    foo: 1,
    bar: 2
  })
  return toRefs(state)
}
export default {
  setup() {
    // 可以在不失去响应式的情况下复用
    const { foo, bar } = useFeatureX()
    return {
      foo,
      bar
    }
  }
}
```
#### 3.4 unref
获取ref值的语法糖
```js
val = isRef(val) ? val.value : val
```
#### 3.5 customRef
创建一个自定义的`ref`,对它的读写进行手动控制。这是文档上的一个防抖的例子
```html
<input v-model="text" />
```
```js
function useDebouncedRef(value, delay = 200) {
  let timeout
  return customRef((track, trigger) => {
    return {
      get() {
        // 读值的时候,触发依赖
        track()
        return value
      },
      set(newValue) {
        clearTimeout(timeout)
        timeout = setTimeout(() => {
          value = newValue
          // 写值的时候，进行防抖，触发依赖
          trigger()
        }, delay)
      }
    }
  })
}

export default {
  setup() {
    return {
      text: useDebouncedRef('hello')
    }
  }
}
```
#### 3.6 shallowRef, triggerRef

shallowRef数据是没有响应式的，但是可以使用triggerRef来手动触发
```js
const shallow = shallowRef({
  greet: 'Hello, world'
})
shallow.value.greet = 'Hello, universe'
triggerRef(shallow) //手动触发响应式
```
### 4. computed and watch
#### 4.1 computed
- 传入一个 getter 函数，返回一个默认不可手动修改的 ref 对象
- 具有 get 和 set 函数的对象来创建可写的 ref 对象
```js
const count = ref(1)
// 只读的
const plusOne = computed(() => count.value + 1)
// 可更改的
const plusOne = computed({
  get: () => count.value + 1,
  set: (val) => {
    count.value = val - 1
  },
})

plusOne.value = 1
console.log(count.value) // 0
```
#### 4.2 watchEffect
立即执行传入的一个函数，并响应式追踪其依赖，并在其依赖变更时重新运行该函数。
```js
const count = ref(0)
watchEffect(() => console.log(count.value))
// 打印出 0,初始化的时候就会打印
setTimeout(() => {
  count.value++
  // -> 打印出 1
}, 100)
```
**停止观察**

卸载组件的时候会自动停止，也可以手动停止监听
```js
const stop = watchEffect(() => {
  /* ... */
})
// 停止监听程序
stop()
```
**副作用**
>如果一个函数内外有依赖于外部变量或者环境时，常常我们称之为其有副作用，如果我们仅通过函数签名不打开内部代码检查并不能知道该函数在干什么，作为一个独立函数我们期望有明确的输入和输出，副作用是bug的发源地，作为程序员开发者应尽量少的开发有副作用的函数或方法，副作用也使得方法通用性下降不适合扩展和可重用性

**运行时机**

```js
// 初始化运行需要读取dom
onMounted(() => {
  watchEffect(() => {
  })
})
//设置运行时机
watchEffect(
  () => {/* ... */},
  {flush: 'pre'}
)
// pre是默认，组件更新前执行
// post, 组件更新后执行
// sync, 同步运行
```
#### 4.3 watch
跟vue2的watch一致
```js
// 侦听一个 getter
const state = reactive({ count: 0 })
watch(
  () => state.count,
  (count, prevCount) => {
    /* ... */
  }
)

// 直接侦听一个 ref
const count = ref(0)
watch(count, (count, prevCount) => {
  /* ... */
})
```
**监听多个数据源**
```js
watch([fooRef, barRef], ([foo, bar], [prevFoo, prevBar]) => {
  /* ... */
})
```
**与 watchEffect 共享的行为**

停止监听，清除副作用，副作用刷新时机，侦听器调试方面跟watchEffect参数是一样的。
## 四.应用配置（原全局配置和全局API）
**注意：全局配置的Vue都用现在的createApp生成的实例来替代**
### createApp(新增)
```js
// vue2挂载dom方法
new Vue({
  render: (h) => h(App)
}).$mount("#app");
// vue3
createApp(App).mount('#app')
```
注意，之前用到的全局配置的Vue都用现在的createApp生成的实例来替代
```js
import { createApp } from 'vue'
const app = createApp({})

// 比如注册全局组件
app.component('my-component', {
  /* ... */
})
// 是否允许devtools检查代码
app.config.devtools = true  
```
### globalProperties(新增)
```js
app.config.globalProperties.foo = 'bar'
app.component('child-component', {
  mounted() {
    console.log(this.foo) // 'bar'
  }
})
```
- 添加可在程序内的任何组件实例中访问的全局属性。当存在键冲突时，组件属性将优先
- 替代掉Vue2.x的 Vue.prototype属性放到原型上的写法
### unmount(新增)
卸载DOM元素上应用程序实例的根组件
```js
const app = createApp(App)
app.mount('#app')
// 5秒之后，卸载APP组件
setTimeout(() => app.unmount('#app'), 5000)
```
## 五. 全局API（新增）
### createApp
类似之前全局的Vue变量，整个组件树共享的上下文，第二个参数作为props值传入
### h
就是之前的createElement。

jsx偶尔在用，但是这个底层的相对用的比较少，就不细讲了
```js
render() {
  return Vue.h('h1', {}, 'Some title')
}
```
### defineComponent（组件）
创建一个组件
```js
import { defineComponent } from 'vue'
const MyComponent = defineComponent({
  data() {
    return { count: 1 }
  },
  methods: {
    increment() {
      this.count++
    }
  }
})
```
### defineAsyncComponent(异步组件)
创建一个异步组件
```js
import { defineAsyncComponent } from 'vue'

const AsyncComp = defineAsyncComponent(() =>
   /*或者*/
  import('./components/AsyncComponent.vue')
   /*或者*/
  new Promise((resolve, reject) => {
  /*可以reject*/
      resolve({
        template: '<div>I am async!</div>'
      })
    })
)

app.component('async-component', AsyncComp)
```
### resolveComponent
只能在`render`和`setup`中使用，通过名称来寻找组件
```js
app.component('MyComponent', {
  /* ... */
})
const MyComponent = resolveComponent('MyComponent')
```
### resolveDynamicComponent
只能在`render`和`setup`中使用，解析活动的组件active
### resolveDirective
只能在`render`和`setup`中使用，允许通过名称解析指令
### withDirectives
只能在`render`和`setup`中使用，允许应用指令到`VNode`。返回一个带有应用指令的`VNode`
```js
const bar = resolveDirective('bar')

return withDirectives(h('div'), [
  [bar, this.y]
])
```
### createRenderer
接受两个泛型参数：`HostNode`和`HostElement`，对应于宿主环境中的`Node`和 `Element`类型。

这个东西不太清楚具体的使用场景

## 六.选项/数据

### emits(新增)
在$emit触发事件之前验证参数
```js
const app = Vue.createApp({})

// 对象语法
app.component('reply-form', {
  created() {
    this.$emit('check')
  },
  emits: {
    // 没有验证函数
    click: null,

    // 带有验证函数
    submit: payload => {
      if (payload.email && payload.password) {
        return true
      } else {
        console.warn(`Invalid submit event payload!`)
        return false
      }
    }
  }
})
```
## 七.选项/DOM(不变)
## 八.选项/生命周期钩子
- ~~beforeDestroy~~ -> `beforeUnmount`
- ~~destroyed~~ -> `unmounted`
- renderTracked（新增）

告诉你哪个操作跟踪了组件以及该操作的目标对象和键
```js
renderTracked({ key, target, type }) {
  // 这里的target为更新之前的值,type是get
  console.log({ key, target, type })
}
```
- renderTriggered（新增）

告诉你是什么操作触发了重新渲染，以及该操作的目标对象和键。
target为更新之后的值,type是set

这些生命周期还可以通过API在setup()里使用
- beforeMount -> `onBeforeMount`
- mounted -> `onMounted`
- beforeUpdate -> `onBeforeUpdate`
- updated -> `onUpdated`
- beforeDestroy -> `onBeforeUnmount`
- destroyed -> `onUnmounted`
- errorCaptured -> `onErrorCaptured`

## 九.选项/资源
### filters(废弃)
过滤器还是有时候在用的

## 十.选项/组合
### parent（废弃）
这个属性平时开发基本不会用到
### setup(详情见上面)

## 十一.选项/其它
### delimiters（废弃）
### functional（废弃）
使用了新的异步组件的生成方法
### model（废弃）
v-model修改了，不需要固定的属性名和事件名了，手动去处理
### comments（废弃）

## 十二.实例property
### vm.$attrs（修改）
现在`$attrs`不但会获取父作用域中不作为组件`props`的值，也可以获取到自定义事件（包含了$listeners的功能）。
### vm.$children（废弃）
### vm.$scopedSlots（废弃）
### vm.$isServer（废弃）
### vm.$listeners（废弃）

## 十三.实例方法/数据
`vue3`底层使用了proxy进行数据监听，所以不需要这两个方法了。
### vm.$set（废弃）
### vm.$delete（废弃）

## 十四.实例方法/事件
因为现在可以直接在setup调用生命周期api，而且可以引入别的js里的方法来使用，所以这些方法也可以不需要了。
### vm.$on（废弃）
### vm.$once（废弃）
### vm.$off（废弃）

## 十五.实例方法/生命周期
### vm.$mount（废弃）
统一使用createApp的mount方法来挂载
### vm.$destroy（废弃）
统一使用createApp的unmount方法来卸载

## 十六.指令
### v-bind（修改）
- ~~.prop~~ 去除
- ~~.sync~~ 去除(现在需要手动去同步)
- .camel 将 kebab-case attribute 名转换为 camelCase
### v-model（修改）
对于组件可以绑定多个属性值
```html
<template>
  <!--在vue3.0中，v-model后面需要跟一个modelValue，即要双向绑定的属性名-->
  <!-- 在Vue3.0中也可以继续使用`Vue2.0`的写法 -->
  <a-input v-model:value="value" placeholder="Basic usage" />
</template>
```

自定义一个组件
```js
<template>
  <div class="custom-input">
    <input :value="value" @input="_handleChangeValue" />
  </div>
</template>
<script>
export default {
  props: {
    value: {
      type: String,
      default: ""
    }
  },
  name: "CustomInput",
  setup(props, { emit }) {
    function _handleChangeValue(e) {
      // vue3.0 是通过emit事件名为 update:modelValue来更新v-model的
      emit("update:value", e.target.value);
    }
    return {
      _handleChangeValue
    };
  }
};
</script>
```
### v-on（修改）
.{keyAlias} - 仅当事件是从特定键触发时才触发回调。不再通过键码的修饰符来触发。

### v-is（新增）
类似vue2中的:is绑定，可以让组件在某些特定的html标签中渲染，比如table,ul。
```html
<!-- 不正确，不会渲染任何内容 -->
<tr v-is="blog-post-row"></tr>
<!-- 正确 -->
<tr v-is="'blog-post-row'"></tr>
```

## 十七.特殊attribute
### key(修改)
循环的时候，`key`要设置在`template`上。
```html
<template v-for="item in list" :key="item.id">
  <div>...</div>
  <span>...</span>
</template>
```
### ref(修改)
`v-for`里使用的`ref`将不会再自动创建数组

解决方式:
```html
<div v-for="item in list" :ref="setItemRef"></div>
```
```js
export default {
  data() {
    return {
      itemRefs: []
    }
  },
  methods: {
    setItemRef(el) {
      this.itemRefs.push(el)
    }
  },
  beforeUpdate() {
    this.itemRefs = []
  },
  updated() {
    console.log(this.itemRefs)
  }
}
```

## 十八.内置的组件
### transition（修改）
- Props新增：
persisted - boolean 如果为true，则表示这是一个转换，实际上不会插入/删除元素，而是切换显示/隐藏状态。 transition 过渡挂钩被注入，但会被渲染器跳过。 相反，自定义指令可以通过调用注入的钩子（例如v-show）来控制过渡

~~enter-class~~ ->`enter-from-class`

~~leave-class~~ ->`leave-from-class`

- 事件
~~before appear~~

### teleport(新增)
将内容插入到目标元素中
```html
<teleport to="#popup" :disabled="displayVideoInline">
  <h1>999999</h1>
</teleport>
```
`to`必填属性，必须是一个有效的query选择器，或者是元素(如果在浏览器环境中使用）。中的内容将会被放置到指定的目标元素中

`disabled`这是一个可选项 ，做一个是可以用来禁用的功能，这意味着它的插槽内容不会移动到任何地方，而是按没有teleport组件一般来呈现【默认为false】

适合场景，全局的loading，多个内容合并等等等。

## 十九.总结

总体来说，`vue3`在尽可能的兼容`vue2`的同时，又引入了全新的组合式API的编程方式，新的这种方式可以有良好的`ts`的支持。

周边的一些组件`vue Router4.0`, `vuex4.0`也都提供了`vue3`支持。组件库这块，`ant design vue`已经支持了`vue3`（对于组件库来说，目前应该也只是基于vue2的基础上只针对一些api的修改去做兼容，完美支持还需重构，现在还不知道有多少坑）。这块内容后续我有时间再写吧。

后面就是一些开发中的比较好的实践了，等过段时间，用`vue3`做具体项目之后再做总结吧。

## 二十.感谢
感谢您的耐心阅读，写文章不易，希望能给我点个赞，谢谢各位。




