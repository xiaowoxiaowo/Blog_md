---
# 主题列表：juejin, github, smartblue, cyanosis, channing-cyan, fancy, hydrogen, condensed-night-purple, greenwillow, v-green, vue-pro
# 贡献主题：https://github.com/xitu/juejin-markdown-themes
theme: smartblue
highlight: vs2015
---

## 前言
这是我最近开发碰到的一个问题，本文是我测试出来的实践结果，供大家参考。

关于js定时器，`setInterval`和`setTimeout`，作为我们日常开发经常使用到的方法，大家一定非常熟悉。比如下面一个例子：
```js
setInterval(() => {
  console.log('1');
}, 500);
```
作为刚学前端没多久的新人也能知道，这段代码就是每过500ms打印一次1（实际运行还需要考虑js的宏任务和微任务的执行时间，定时器的间隔时间是500ms，但是定时器中的方法触发可能需要在宏任务队列中排队，不一定会在500ms的时候触发，关于Event Loop的基础内容不在本文讨论之内）。

但是如果你把浏览器从当前页面切换到另一个标签页，或者把浏览器最小化了，这时候，**这个页面定时器的间隔时间还是500ms**？

本文将测试`setInterval`、`setTimeout`、`requestAnimationFrame`这三个方法在浏览器可见以及不可见状态下的表现，我的测试浏览器以及版本是`谷歌(86.0.4240.193)`,`火狐(81.0.2)`,`ie11`。

## 浏览器可见和不可见状态
浏览器的可见和不可见状态的切换会触发`visibilitychange`事件，我们可以通过监听这个事件来判别浏览器的可见状态。
```js
document.addEventListener("visibilitychange", function() {
  console.log(document.visibilityState);
});
```
`document.visibilityState`有三个值
- hidden：页面彻底不可见。
- visible：页面至少一部分可见。
- prerender：页面即将或正在渲染，处于不可见状态。
这里重点关注`hidden`这个值，当我们浏览器切换当前页面到另外一个标签页或者把浏览器最小化的时候，`document.visibilityState`就会是`hidden`值。我们也可以使用`document.hidden`，它返回一个布尔值，为`true`的时候，说明当前浏览器是不可见状态。

关于`visibilitychange`的细节可以看阮一峰老师的这篇文章 [Page Visibility API 教程](http://www.ruanyifeng.com/blog/2018/10/page_visibility_api.html)。

## setInterval
我们先来测试`setInterval`，代码如下
```js
<button id="btn">开始计时</button>

// 兼容ie写法
document.getElementById('btn').addEventListener('click', function() {
  setInterval(function() {
    const myDate = new Date();
    const currentDate = myDate.getMinutes() + '分'+ myDate.getSeconds() + '秒' + myDate.getMilliseconds() + '豪秒';
    // 每次循环打印当前时间
    console.log(currentDate);
  }, 500);
});

// 浏览器可见状态切换事件
document.addEventListener('visibilitychange', function() { 
  if(document.hidden) {
    console.log('页面不可见');
  }
});
```
定时器间隔是500ms，先来看下`谷歌浏览器`

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d7172bf22a2d426d90e1a38b3f3c43bc~tplv-k3u1fbpfcp-watermark.image)

我们发现，**当页面不可见之后，定时器的间隔变成了1s。** 接下来，我们把定时器间隔改成2s来试下。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2000c53a93f349b692da66d18f3b2c86~tplv-k3u1fbpfcp-watermark.image)

前后间隔时间一致。

接下来测试一下`火狐`和`ie`。这里列出的图片都是500ms和2s的例子。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fe3a21f99fe04f38b95a3384aa4fa930~tplv-k3u1fbpfcp-watermark.image)
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d18df7fcd02b45988882bcd0649cdc99~tplv-k3u1fbpfcp-watermark.image)

ie浏览器

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cd73a2a2c7be4138b51d18fc208420d2~tplv-k3u1fbpfcp-watermark.image)

经过我大量的测试，可以得出结论，**谷歌浏览器中，当页面处于不可见状态时，setInterval的最小间隔时间会被限制为1s。火狐浏览器的setInterval和谷歌特性一致，但是ie浏览器没有对不可见状态时的setInterval进行性能优化，不可见前后间隔时间不变**。

## setTimeout
接下来是`setTimeout`

```js
function timer() {
  setTimeout(function() {
    const myDate = new Date();
    const currentDate = myDate.getMinutes() + '分'+ myDate.getSeconds() + '秒' + myDate.getMilliseconds() + '豪秒';
    console.log(currentDate);
    timer();
  }, 500)
}
// 兼容ie写法
document.getElementById('btn').addEventListener('click', function() {
  timer();
});
```
同样先来看看在谷歌浏览器中的表现（还是500ms和2s）

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6e847d57febf48dfbd09057a7d7bb10b~tplv-k3u1fbpfcp-watermark.image)
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1346d8cd190c4dceb0e62f8cf29c4e51~tplv-k3u1fbpfcp-watermark.image)

我们发现在谷歌浏览器中，500ms的间隔，`setTimeout`和`setInterval`表现一致，都是**最小间隔限制为1s**。但是2s隔间的测试结果出现了分歧，页面不可见之后，间隔变成了3s。继续经过多次的测试，如下，左图的间隔时间为990ms，右图的间隔时间为1s。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a87bdef96b9f4008aac7c7325685ae95~tplv-k3u1fbpfcp-watermark.image)
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9e07ffe4deff471082228bac1a4cfe12~tplv-k3u1fbpfcp-watermark.image)

不可见状态下，左图中的990ms间隔时间变为1s，右图中的1s间隔时间变为2s。


我们再来看看`火狐`（500ms和2s）

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1bae222008d54f56ac87dfdcbed6b069~tplv-k3u1fbpfcp-watermark.image)
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5a6b9a2a658c4d45bef59f9ac6baba5f~tplv-k3u1fbpfcp-watermark.image)

`火狐`浏览器不可见状态下,左图中的500ms变为1s，右图中的2s保持不变。

再来看看`ie`浏览器(500ms)

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/967b40e09120432db8f1bd211b486711~tplv-k3u1fbpfcp-watermark.image)

一样毫无优化。

我们可以得出结论，**在谷歌浏览器中，setTimeout在浏览器不可见状态下间隔低于1s的会变为1s，大于等于1s的会变成N+1s的间隔值。火狐浏览器下setTimeout的最小间隔时间会变为1s,大于等于1s的间隔不变。ie浏览器在不可见状态前后的间隔时间不变。**

## requestAnimationFrame
raf是浏览器提供的一个更流畅的处理动画的方法，它会在下次浏览器GUI绘制页面的时候运行传入的方法。GUI绘制页面的频率跟显示器的刷新率有关，普通显示器的刷新率是60hz，因此raf在一秒之内需要运行60次，间隔四舍五入大概是17ms。

```js
function timer() {
  const myDate = new Date();
  const currentDate = myDate.getMinutes() + '分'+ myDate.getSeconds() + '秒' + myDate.getMilliseconds() + '豪秒';
  console.log(currentDate);
  window.requestAnimationFrame(timer)
}
// 兼容ie写法
document.getElementById('btn').addEventListener('click', function() {
  timer();
});
```
我们来看看不同浏览器下面的表现：

`谷歌浏览器`

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dda4b3b6f82149dd9b5abde2d0475a1f~tplv-k3u1fbpfcp-watermark.image)

`火狐浏览器`

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b6e0d2f482d45228cdbe66a13c8e7aa~tplv-k3u1fbpfcp-watermark.image)

`ie浏览器`

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3987bf181614046a6b458a2647e1f97~tplv-k3u1fbpfcp-watermark.image)

我们可以发现，**谷歌浏览器和ie浏览器当浏览器状态为不可见时，raf方法将停止执行。火狐浏览器当状态变为不可见时，会在间隔是1s,2s,4s,8s,16s,32s...这样的顺序下去执行raf方法。**

## 总结
谷歌浏览器中，当页面处于不可见状态时，`setInterval`的最小间隔时间会被限制为1s。火狐浏览器的`setInterval`和谷歌特性一致。ie浏览器没有对不可见状态时的`setInterval`进行性能优化，不可见前后间隔时间不变。

在谷歌浏览器中，`setTimeout`在浏览器不可见状态下间隔低于1s的会变为1s，大于等于1s的会变成N+1s的间隔值。火狐浏览器下`setTimeout`的最小间隔时间会变为1s,大于等于1s的间隔不变。ie浏览器在不可见状态前后的间隔时间不变。

谷歌浏览器和ie浏览器当浏览器状态为不可见时，`raf`方法将停止执行。火狐浏览器当状态变为不可见时，会在间隔是1s,2s,4s,8s,16s,32s...这样的顺序下去执行`raf`方法。

## 如何解决
碰到问题当然需要解决，在一些定时器小于1s的倒计时的页面中，如果用户切换到了其他标签页。再切回去的时候，页面上显示的倒计时时间其实是错误的，这种隐藏的bug会带来很大的风险。该怎么解决呢？

除了调取后台接口或者`websocket`连接之外，其实有一个更好的解决方案，`webWorkers`。而且`webWorkers`还可以解决一个页面存在多个定时器时候间隔时间误差较大的问题。

直接上例子
```js
document.getElementById('btn').addEventListener('click', function() {
  var w = new Worker('demo_workers.js');
  w.onmessage = function(event){
    console.log(event.data);
  };
});
//浏览器切换事件
document.addEventListener('visibilitychange', function() { 
  if(document.hidden) {
    console.log('页面不可见');
  }
});
```
```js
// demo_workers.js
setInterval(function() {
  const myDate = new Date();
  const currentDate = myDate.getMinutes() + '分'+ myDate.getSeconds() + '秒' + myDate.getMilliseconds() + '豪秒';
  postMessage(currentDate);
}, 500);
```
实际结果

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9684bb9f52964d528bcc18f58f09ed6b~tplv-k3u1fbpfcp-watermark.image)
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5932a2da9eca44f2adfb0e6beb295a6d~tplv-k3u1fbpfcp-watermark.image)

间隔保持一致。

## 感谢
如果本文有帮助到你的地方，请帮忙点个赞，感谢支持。
