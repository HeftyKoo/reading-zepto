# 读Zepto源码之Fx模块

`fx` 模块为利用 `CSS3` 的过渡和动画的属性为 `Zepto` 提供了动画的功能，在 `fx` 模块中，只做了事件和样式浏览器前缀的补全，没有做太多的兼容。对于不支持 `CSS3` 过渡和动画的， `Zepto` 的处理也相对简单，动画立即完成，马上执行回调。

读 Zepto 源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## GitBook

《[reading-zepto](https://yeyuqiudeng.gitbooks.io/reading-zepto/content/)》

## 内部方法

### dasherize

```javascript
function dasherize(str) { return str.replace(/([A-Z])/g, '-$1').toLowerCase() }
```

这个方法是将驼峰式（ `camleCase` ）的写法转换成用 `-` 连接的连词符的写法（ `camle-case` ）。转换的目的是让写法符合 `css` 的样式规范。

### normalizeEvent

```javascript
function normalizeEvent(name) { return eventPrefix ? eventPrefix + name : name.toLowerCase() }
```

为事件名增加浏览器前缀。

## 为事件和样式增加浏览器前缀

### 变量

```javascript
var prefix = '', eventPrefix,
    vendors = { Webkit: 'webkit', Moz: '', O: 'o' },
    testEl = document.createElement('div'),
    supportedTransforms = /^((translate|rotate|scale)(X|Y|Z|3d)?|matrix(3d)?|perspective|skew(X|Y)?)$/i,
    transform,
    transitionProperty, transitionDuration, transitionTiming, transitionDelay,
    animationName, animationDuration, animationTiming, animationDelay,
    cssReset = {}
```

`vendors` 定义了浏览器的样式前缀（ `key` ） 和事件前缀 ( `value` ) 。

`testEl` 是为检测浏览器前缀所创建的临时节点。

`cssReset` 用来保存加完前缀后的样式规则，用来过渡或动画完成后重置样式。

### 浏览器前缀检测

```javascript
if (testEl.style.transform === undefined) $.each(vendors, function(vendor, event){
  if (testEl.style[vendor + 'TransitionProperty'] !== undefined) {
    prefix = '-' + vendor.toLowerCase() + '-'
    eventPrefix = event
    return false
  }
})
```

检测到浏览器不支持标准的 `transform` 属性，则依次检测加了不同浏览器前缀的 `transitionProperty` 属性，直至找到合适的浏览器前缀，样式前缀保存在 `prefix` 中， 事件前缀保存在 `eventPrefix` 中。

### 初始化样式

```javascript
transform = prefix + 'transform'
cssReset[transitionProperty = prefix + 'transition-property'] =
cssReset[transitionDuration = prefix + 'transition-duration'] =
cssReset[transitionDelay    = prefix + 'transition-delay'] =
cssReset[transitionTiming   = prefix + 'transition-timing-function'] =
cssReset[animationName      = prefix + 'animation-name'] =
cssReset[animationDuration  = prefix + 'animation-duration'] =
cssReset[animationDelay     = prefix + 'animation-delay'] =
cssReset[animationTiming    = prefix + 'animation-timing-function'] = ''

```

获取浏览器前缀后，为所有的 `transition` 和 `animation` 属性加上对应的前缀，都初始化为 `''`，方便后面使用。

## 方法

### $.fx

```javascript
$.fx = {
  off: (eventPrefix === undefined && testEl.style.transitionProperty === undefined),
  speeds: { _default: 400, fast: 200, slow: 600 },
  cssPrefix: prefix,
  transitionEnd: normalizeEvent('TransitionEnd'),
  animationEnd: normalizeEvent('AnimationEnd')
}
```

* off:  表示浏览器是否支持过渡或动画，如果既没有浏览器前缀，也不支持标准的属性，则判定该浏览器不支持动画
* speeds: 定义了三种动画持续的时间， 默认为 `400ms` 
* cssPrefix: 样式浏览器兼容前缀，即 `prefix`
* transitionEnd： 过渡完成时触发的事件，调用 `normalizeEvent` 事件加了浏览器前缀补全
* animationEnd： 动画完成时触发的事件，同样加了浏览器前缀补全

### animate

```javascript
$.fn.animate = function(properties, duration, ease, callback, delay){
  if ($.isFunction(duration))
    callback = duration, ease = undefined, duration = undefined
  if ($.isFunction(ease))
    callback = ease, ease = undefined
  if ($.isPlainObject(duration))
    ease = duration.easing, callback = duration.complete, delay = duration.delay, duration = duration.duration
  if (duration) duration = (typeof duration == 'number' ? duration :
                            ($.fx.speeds[duration] || $.fx.speeds._default)) / 1000
  if (delay) delay = parseFloat(delay) / 1000
  return this.anim(properties, duration, ease, callback, delay)
}
```

我们平时用得最多的是 `animate` 这个方法，但是这个方法最终调用的是 `anim` 这个方法，`animate` 这个方法相当灵活，因为它主要做的是参数修正的工作，做得参数适应 `anim` 的接口。

#### 参数：

* properties：需要过渡的样式对象，或者 `animation` 的名称，只有这个参数是必传的
* duration: 过渡时间
* ease: 缓动函数
* callback: 过渡或者动画完成后的回调函数
* delay: 过渡或动画延迟执行的时间

#### 修正参数

```javascript
if ($.isFunction(duration))
  callback = duration, ease = undefined, duration = undefined
```

这是处理传参为 `animate(properties, callback)` 的情况。

```javascript
if ($.isFunction(ease))
    callback = ease, ease = undefined
```

这是处理 `animate(properties, duration, callback)` 的情况，此时 `callback` 在参数 `ease` 的位置

```javascript
if ($.isPlainObject(duration))
  ease = duration.easing, callback = duration.complete, delay = duration.delay, duration = duration.duration
```

这是处理 `animate(properties, { duration: msec, easing: type, complete: fn }) ` 的情况。除了 `properties` ，后面的参数还可以写在一个对象中传入。

如果检测到为对象的传参方式，则将对应的值从对象中取出。

```javascript
if (duration) duration = (typeof duration == 'number' ? duration :
                          ($.fx.speeds[duration] || $.fx.speeds._default)) / 1000
```

如果过渡时间为数字，则直接采用，如果是 `speeds` 中指定的 `key` ，即 `slow` 、`fast` 甚至 `_default` ，则从 `speeds` 中取值，否则用 `speends` 的 `_default` 值。

因为在样式中是用 `s` 取值，所以要将毫秒数除 `1000`。

```javascript
if (delay) delay = parseFloat(delay) / 1000
```

也将延迟时间转换为秒。

### anim

```javascript
$.fn.anim = function(properties, duration, ease, callback, delay){
  var key, cssValues = {}, cssProperties, transforms = '',
      that = this, wrappedCallback, endEvent = $.fx.transitionEnd,
      fired = false

  if (duration === undefined) duration = $.fx.speeds._default / 1000
  if (delay === undefined) delay = 0
  if ($.fx.off) duration = 0

  if (typeof properties == 'string') {
    // keyframe animation
    cssValues[animationName] = properties
    cssValues[animationDuration] = duration + 's'
    cssValues[animationDelay] = delay + 's'
    cssValues[animationTiming] = (ease || 'linear')
    endEvent = $.fx.animationEnd
  } else {
    cssProperties = []
    // CSS transitions
    for (key in properties)
      if (supportedTransforms.test(key)) transforms += key + '(' + properties[key] + ') '
    else cssValues[key] = properties[key], cssProperties.push(dasherize(key))

    if (transforms) cssValues[transform] = transforms, cssProperties.push(transform)
    if (duration > 0 && typeof properties === 'object') {
      cssValues[transitionProperty] = cssProperties.join(', ')
      cssValues[transitionDuration] = duration + 's'
      cssValues[transitionDelay] = delay + 's'
      cssValues[transitionTiming] = (ease || 'linear')
    }
  }

  wrappedCallback = function(event){
    if (typeof event !== 'undefined') {
      if (event.target !== event.currentTarget) return // makes sure the event didn't bubble from "below"
      $(event.target).unbind(endEvent, wrappedCallback)
    } else
      $(this).unbind(endEvent, wrappedCallback) // triggered by setTimeout

    fired = true
    $(this).css(cssReset)
    callback && callback.call(this)
  }
  if (duration > 0){
    this.bind(endEvent, wrappedCallback)
    // transitionEnd is not always firing on older Android phones
    // so make sure it gets fired
    setTimeout(function(){
      if (fired) return
      wrappedCallback.call(that)
    }, ((duration + delay) * 1000) + 25)
  }

  // trigger page reflow so new elements can animate
  this.size() && this.get(0).clientLeft

  this.css(cssValues)

  if (duration <= 0) setTimeout(function() {
    that.each(function(){ wrappedCallback.call(this) })
  }, 0)

  return this
}
```

`animation` 最终调用的是 `anim` 方法，`Zepto` 也将这个方法暴露了出去，其实我觉得只提供 `animation` 方法就可以了，这个方法完全可以作为私有的方法调用。

#### 参数默认值

```javascript
if (duration === undefined) duration = $.fx.speeds._default / 1000
if (delay === undefined) delay = 0
if ($.fx.off) duration = 0
```

如果没有传递持续时间 `duration` ，则默认为 `$.fx.speends._default` 的定义值 `400ms` ，这里需要转换成 `s` 。

如果没有传递 `delay` ，则默认不延迟，即 `0` 。

如果浏览器不支持过渡和动画，则 `duration` 设置为 `0` ，即没有动画，立即执行回调。

#### 处理animation动画参数

```javascript
if (typeof properties == 'string') {
  // keyframe animation
  cssValues[animationName] = properties
  cssValues[animationDuration] = duration + 's'
  cssValues[animationDelay] = delay + 's'
  cssValues[animationTiming] = (ease || 'linear')
  endEvent = $.fx.animationEnd
} 
```

如果 `properties` 为 `string`， 即 `properties` 为动画名，则设置动画对应的 `css` ，`duration` 和 `delay` 都加上了 `s` 的单位，默认的缓动函数为 `linear` 。

#### 处理transition参数

````javascript
else {
  cssProperties = []
  // CSS transitions
  for (key in properties)
    if (supportedTransforms.test(key)) transforms += key + '(' + properties[key] + ') '
  else cssValues[key] = properties[key], cssProperties.push(dasherize(key))

  if (transforms) cssValues[transform] = transforms, cssProperties.push(transform)
  if (duration > 0 && typeof properties === 'object') {
    cssValues[transitionProperty] = cssProperties.join(', ')
    cssValues[transitionDuration] = duration + 's'
    cssValues[transitionDelay] = delay + 's'
    cssValues[transitionTiming] = (ease || 'linear')
  }
}
````

`supportedTransforms` 是用来检测是否为 `transform` 的正则，如果是 `transform` ，则拼接成符合 `transform` 规则的字符串。

否则，直接将值存入 `cssValues` 中，将 `css` 的样式名存入 `cssProperties` 中，并且调用了 `dasherize` 方法，使得 `properties`  的 `css` 样式名（ `key` ）支持驼峰式的写法。

```javascript
if (transforms) cssValues[transform] = transforms, cssProperties.push(transform)
```

这段是检测是否有 `transform`  ，如果有，也将 `transform` 存入 `cssValues` 和 `cssProperties` 中。

接下来判断动画是否开启，并且是否有过渡属性，如果有，则设置对应的值。

#### 回调函数的处理

```javascript
wrappedCallback = function(event){
  if (typeof event !== 'undefined') {
    if (event.target !== event.currentTarget) return // makes sure the event didn't bubble from "below"
    $(event.target).unbind(endEvent, wrappedCallback)
  } else
    $(this).unbind(endEvent, wrappedCallback) // triggered by setTimeout

  fired = true
  $(this).css(cssReset)
  callback && callback.call(this)
}
```

如果浏览器支持过渡或者动画事件，则在动画结束的时候，取消事件监听，注意在 `unbind` 时，有个 `event.target !== event.currentTarget` 的判定，这是排除冒泡事件。

如果事件不存在时，直接取消对应元素上的事件监听。

并且将状态控制 `fired` 设置为 `true` ，表示回调已经执行。

动画完成后，再将涉及过渡或动画的样式设置为空。

最后，调用传递进来的回调函数，整个动画完成。

#### 绑定过渡或动画的结束事件

```javascript
if (duration > 0){
  this.bind(endEvent, wrappedCallback)
  setTimeout(function(){
    if (fired) return
    wrappedCallback.call(that)
  }, ((duration + delay) * 1000) + 25)
}
```

绑定过渡或动画的结束事件，在动画结束时，执行处理过的回调函数。

注意这里有个 `setTimeout` ，是避免浏览器不支持过渡或动画事件时，可以通过 `setTimeout` 执行回调。`setTimeout` 的回调执行比动画时间长 `25ms` ，目的是让事件响应在 `setTimeout` 之前，如果浏览器支持过渡或动画事件， `fired` 会在回调执行时设置成 `true`， `setTimeout` 的回调函数不会再重复执行。

#### 触发页面回流

```javascript
 // trigger page reflow so new elements can animate
this.size() && this.get(0).clientLeft

this.css(cssValues)

```

这里用了点黑科技，读取 `clientLeft` 属性，触发页面的回流，使得动画的样式设置上去时可以立即执行。

具体可以这篇文章中的解释：[2014-02-07-hidden-documentation.md](https://github.com/mislav/blog/blob/master/_posts/2014-02-07-hidden-documentation.md) 

#### 过渡时间不大于零的回调处理

```javascript
if (duration <= 0) setTimeout(function() {
  that.each(function(){ wrappedCallback.call(this) })
}, 0)
```

`duration` 不大于零时，可以是参数设置错误，也可能是浏览器不支持过渡或动画，就立即执行回调函数。

## 系列文章

1. [读Zepto源码之代码结构](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E4%BB%A3%E7%A0%81%E7%BB%93%E6%9E%84.md)
2. [读Zepto源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md)
3. [读Zepto源码之工具函数](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md)
4. [读Zepto源码之神奇的$](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E7%A5%9E%E5%A5%87%E7%9A%84%24.md)
5. [读Zepto源码之集合操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E6%93%8D%E4%BD%9C.md)
6. [读Zepto源码之集合元素查找](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E5%85%83%E7%B4%A0%E6%9F%A5%E6%89%BE.md)
7. [读Zepto源码之操作DOM](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E6%93%8D%E4%BD%9CDOM.md)
8. [读Zepto源码之样式操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E6%A0%B7%E5%BC%8F%E6%93%8D%E4%BD%9C.md)
9. [读Zepto源码之属性操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B1%9E%E6%80%A7%E6%93%8D%E4%BD%9C.md)
10. [读Zepto源码之Event模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BEvent%E6%A8%A1%E5%9D%97.md)
11. [读Zepto源码之IE模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BIE%E6%A8%A1%E5%9D%97.md)
12. [读Zepto源码之Callbacks模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BCallbacks%E6%A8%A1%E5%9D%97.md)
13. [读Zepto源码之Deferred模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BDeferred%E6%A8%A1%E5%9D%97.md)
14. [读Zepto源码之Ajax模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BAjax%E6%A8%A1%E5%9D%97.md)
15. [读Zepto源码之Assets模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8Bassets%E6%A8%A1%E5%9D%97.md)
16. [读Zepto源码之Selector模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BSelector%E6%A8%A1%E5%9D%97.md)
17. [读Zepto源码之Touch模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BTouch%E6%A8%A1%E5%9D%97.md)
18. [读Zepto源码之Gesture模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BGesture%E6%A8%A1%E5%9D%97.md)
19. [读Zepto源码之IOS3模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BIOS3%E6%A8%A1%E5%9D%97.md)


### 附文

* [译：怎样处理 Safari 移动端对图片资源的限制](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E9%99%84%EF%BC%9A%E6%80%8E%E6%A0%B7%E5%A4%84%E7%90%86%20Safari%20%E7%A7%BB%E5%8A%A8%E7%AB%AF%E5%AF%B9%E5%9B%BE%E7%89%87%E8%B5%84%E6%BA%90%E7%9A%84%E9%99%90%E5%88%B6.md)



## 参考

* [一步一步DIY zepto库，研究zepto源码7--动画模块(fx fx_method)](https://zrysmt.github.io/2017/04/28/%E4%B8%80%E6%AD%A5%E4%B8%80%E6%AD%A5DIY%20zepto%E5%BA%93%EF%BC%8C%E7%A0%94%E7%A9%B6zepto%E6%BA%90%E7%A0%817--%E5%8A%A8%E7%94%BB%E6%A8%A1%E5%9D%97(fx%EF%BC%8Cfx_method)/)
* [How (not) to trigger a layout in WebKit](http://gent.ilcore.com/2011/03/how-not-to-trigger-layout-in-webkit.html)
* [2014-02-07-hidden-documentation.md](https://github.com/mislav/blog/blob/master/_posts/2014-02-07-hidden-documentation.md)

## License

[署名-非商业性使用-禁止演绎 4.0 国际 (CC BY-NC-ND 4.0)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://user-gold-cdn.xitu.io/2017/5/30/76626b0be42083d36b36f4a117dc1873) 

作者：对角另一面