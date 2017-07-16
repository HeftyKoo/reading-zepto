# 读Zepto源码之Event模块

Event 模块是 Zepto 必备的模块之一，由于对 Event Api 不太熟，Event 对象也比较复杂，所以乍一看 Event 模块的源码，有点懵，细看下去，其实也不太复杂。

读Zepto源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## 准备知识

### focus/blur 的事件模拟

为什么要对 `focus` 和 `blur` 事件进行模拟呢？从 MDN 中可以看到， `focus` 事件和 `blur` 事件并不支持事件冒泡。不支持事件冒泡带来的直接后果是不能进行事件委托，所以需要对 `focus` 和 `blur` 事件进行模拟。

除了 `focus` 事件和 `blur` 事件外，现代浏览器还支持 `focusin` 事件和 `focusout` 事件，他们和 `focus` 事件及 `blur` 事件的最主要区别是支持事件冒泡。因此可以用 `focusin` 和模拟 `focus` 事件的冒泡行为，用 `focusout` 事件来模拟 `blur` 事件的冒泡行为。

我们可以通过以下代码来确定这四个事件的执行顺序：

```html
<input id="test" type="text" />
```

```javascript
const target = document.getElementById('test')
target.addEventListener('focusin', () => {console.log('focusin')})
target.addEventListener('focus', () => {console.log('focus')})
target.addEventListener('blur', () => {console.log('blur')})
target.addEventListener('focusout', () => {console.log('focusout')})
```

在 `chrome59`下， `input` 聚焦和失焦时，控制台会打印出如下结果：

```javascript
'focus'
'focusin'
'blur'
'focusout'
```

可以看到，在此浏览器中，事件的执行顺序应该是 `focus > focusin > blur > focusout`

关于这几个事件更详细的描述，可以查看：《[说说focus /focusin /focusout /blur 事件](https://segmentfault.com/a/1190000003942014)》

关于事件的执行顺序，我测试的结果与文章所说的有点不太一样。感兴趣的可以点击这个链接测试下[http://jsbin.com/nizugazamo/edit?html,js,console,output](http://jsbin.com/nizugazamo/edit?html,js,console,output)。不过我觉得执行顺序可以不必细究，可以将 `focusin` 作为 `focus` 事件的冒泡版本。

### mouseenter/mouseleave 的事件模拟

跟 `focus` 和 `blur` 一样，`mouseenter` 和 `mouseleave` 也不支持事件的冒泡， 但是 `mouseover` 和 `mouseout` 支持事件冒泡，因此，这两个事件的冒泡处理也可以分别用 `mouseover` 和 `mouseout` 来模拟。

在鼠标事件的 `event` 对象中，有一个 `relatedTarget` 的属性，从 [MDN:MouseEvent.relatedTarget](https://developer.mozilla.org/en-US/docs/Web/API/MouseEvent/relatedTarget) 文档中，可以看到，`mouseover` 的 `relatedTarget` 指向的是移到目标节点上时所离开的节点（ `exited from` ），`mouseout` 的 `relatedTarget` 所指向的是离开所在的节点后所进入的节点（ ` entered to` ）。

另外 `mouseover` 事件会随着鼠标的移动不断触发，但是 `mouseenter` 事件只会在进入节点的那一刻触发一次。如果鼠标已经在目标节点上，那 `mouseover` 事件触发时的 `relatedTarget` 为当前节点。

因此，要模拟 `mouseenter` 或 `mouseleave` 事件，只需要确定触发 `mouseover` 或 `mouseout` 事件上的 `relatedTarget` 不存在，或者 `relatedTarget` 不为当前节点，并且不为当前节点的子节点，避免子节点事件冒泡的影响。

关于 `mouseenter` 和 `mouseleave` 的模拟， [谦龙](https://github.com/qianlongo) 有篇文章《[mouseenter与mouseover为何这般纠缠不清？](https://juejin.im/post/5935773fa0bb9f0058edbd61)》写得很清楚，建议读一下。

## Event 模块的核心

将 `Event` 模块简化后如下：

```javascript
;(function($){})(Zepto)
```

其实就是向闭包中传入 `Zepto` 对象，然后对 `Zepto` 对象做一些扩展。

在 `Event` 模块中，主要做了如下几件事：

* 提供简洁的API
* 统一不同浏览器的 `event` 对象
* 事件句柄缓存池，方便手动触发事件和解绑事件。
* 事件委托

## 内部方法

### zid

```javascript
var _zid = 1
function zid(element) {
  return element._zid || (element._zid = _zid++)
}
```

获取参数 `element` 对象的 `_zid` 属性，如果属性不存在，则全局变量 `_zid` 增加 `1` ，作为 `element` 的 `_zid` 的属性值返回。这个方法用来标记已经绑定过事件的元素，方便查找。

### parse

```javascript
function parse(event) {
  var parts = ('' + event).split('.')
  return {e: parts[0], ns: parts.slice(1).sort().join(' ')}
}
```

在 `zepto` 中，支持事件的命名空间，可以用 `eventType.ns1.ns2...` 的形式来给事件添加一个或多个命名空间。

`parse` 函数用来分解事件名和命名空间。

`'' + event` 是将 `event` 变成字符串，再以 `.` 分割成数组。

返回的对象中，`e` 为事件名， `ns` 为排序后，以空格相连的命名空间字符串，形如 `ns1 ns2 ns3 ...` 的形式。

### matcherFor

```javascript
function matcherFor(ns) {
  return new RegExp('(?:^| )' + ns.replace(' ', ' .* ?') + '(?: |$)')
}
```

生成匹配命名空间的表达式，例如，传进来的参数 `ns` 为 `ns1 ns2 ns3` ，最终生成的正则为 `/(?:^| )ns1.* ?ns2.* ?ns3(?: |$)/`。至于有什么用，下面马上讲到。

### findHandlers，查找缓存的句柄

```javascript
handlers = {}
function findHandlers(element, event, fn, selector) {
  event = parse(event)
  if (event.ns) var matcher = matcherFor(event.ns)
  return (handlers[zid(element)] || []).filter(function(handler) {
    return handler
      && (!event.e  || handler.e == event.e)
      && (!event.ns || matcher.test(handler.ns))
      && (!fn       || zid(handler.fn) === zid(fn))
      && (!selector || handler.sel == selector)
  })
}
```

查找元素对应的事件句柄。

```javascript
event = parse(event)
```

调用 `parse` 函数，分隔出 `event` 参数的事件名和命名空间。

```javascript
if (event.ns) var matcher = matcherFor(event.ns)
```

如果命名空间存在，则生成匹配该命名空间的正则表达式 `matcher`。

```javascript
return (handlers[zid(element)] || []).filter(function(handler) {
    ...
  })
```

返回的其实是 `handlers[zid(element)]` 中符合条件的句柄函数。 `handlers` 是缓存的句柄容器，用 `element` 的 `_zid` 属性值作为 `key` 。

 ```javascript
return handler  // 条件1
      && (!event.e  || handler.e == event.e) // 条件2
      && (!event.ns || matcher.test(handler.ns)) // 条件3
      && (!fn       || zid(handler.fn) === zid(fn)) // 条件4
      && (!selector || handler.sel == selector) // 条件5
 ```

返回的句柄必须满足5个条件：

1. 句柄必须存在
2. 如果 `event.e` 存在，则句柄的事件名必须与 `event` 的事件名一致
3. 如果命名空间存在，则句柄的命名空间必须要与事件的命名空间匹配（ `matcherFor` 的作用 ）
4. 如果指定匹配的事件句柄为 `fn` ，则当前句柄 `handler` 的 `_zid` 必须与指定的句柄 `fn` 相一致
5. 如果指定选择器 `selector` ，则当前句柄中的选择器必须与指定的选择器一致

从上面的比较可以看到，缓存的句柄对象的形式如下：

```javascript
{
  fn: '', // 函数
  e: '', // 事件名
  ns: '', // 命名空间
  sel: '',  // 选择器
  // 除此之外，其实还有
  i: '', // 函数索引
  del: '', // 委托函数
  proxy: '', // 代理函数
  // 后面这几个属性会讲到
}
```

### realEvent，返回对应的冒泡事件

```javascript
focusinSupported = 'onfocusin' in window,
focus = { focus: 'focusin', blur: 'focusout' },
hover = { mouseenter: 'mouseover', mouseleave: 'mouseout' }
function realEvent(type) {
  return hover[type] || (focusinSupported && focus[type]) || type
}
```

这个函数其实是将 `focus/blur` 转换成 `focusin/focusout` ，将 `mouseenter/mouseleave` 转换成 `mouseover/mouseout` 事件。

由于 `focusin/focusout` 事件浏览器支持程度还不是很好，因此要对浏览器支持做一个检测，如果浏览器支持，则返回，否则，返回原事件名。

### compatible，修正event对象

```javascript
returnTrue = function(){return true},
returnFalse = function(){return false},
eventMethods = {
  preventDefault: 'isDefaultPrevented',
  stopImmediatePropagation: 'isImmediatePropagationStopped',
  stopPropagation: 'isPropagationStopped'
}

function compatible(event, source) {
  if (source || !event.isDefaultPrevented) {
    source || (source = event)

    $.each(eventMethods, function(name, predicate) {
      var sourceMethod = source[name]
      event[name] = function(){
        this[predicate] = returnTrue
        return sourceMethod && sourceMethod.apply(source, arguments)
      }
      event[predicate] = returnFalse
    })

    try {
      event.timeStamp || (event.timeStamp = Date.now())
    } catch (ignored) { }

    if (source.defaultPrevented !== undefined ? source.defaultPrevented :
        'returnValue' in source ? source.returnValue === false :
        source.getPreventDefault && source.getPreventDefault())
      event.isDefaultPrevented = returnTrue
      }
  return event
}
```

`compatible` 函数用来修正 `event` 对象的浏览器差异，向 `event` 对象中添加了 `isDefaultPrevented、isImmediatePropagationStopped、isPropagationStopped` 几个方法，对不支持 `timeStamp` 的浏览器，向 `event` 对象中添加 `timeStamp` 属性。

```javascript
if (source || !event.isDefaultPrevented) {
  source || (source = event)

  $.each(eventMethods, function(name, predicate) {
    var sourceMethod = source[name]
    event[name] = function(){
      this[predicate] = returnTrue
      return sourceMethod && sourceMethod.apply(source, arguments)
    }
    event[predicate] = returnFalse
  })
```

判断条件是，原事件对象存在，或者事件 `event` 的 `isDefaultPrevented` 不存在时成立。

如果 `source` 不存在，则将 `event` 赋值给 `source`， 作为原事件对象。

遍历 `eventMethods` ，获得原事件对象的对应方法名 `sourceMethod`。

```javascript
event[name] = function(){
  this[predicate] = returnTrue
  return sourceMethod && sourceMethod.apply(source, arguments)
}
```

改写 `event` 对象相应的方法，如果执行对应的方法时，先将事件中方法所对应的新方法赋值为 `returnTrue` 函数 ，例如执行 `preventDefault` 方法时， `isDefaultPrevented` 方法的返回值为 `true`。

```javascript
event[predicate] = returnFalse
```

这是将新添加的属性，初始化为 `returnFalse` 方法

```javascript
try {
  event.timeStamp || (event.timeStamp = Date.now())
} catch (ignored) { }
```

这段向不支持 `timeStamp` 属性的浏览器中添加 `timeStamp` 属性。

```javascript
if (source.defaultPrevented !== undefined ? source.defaultPrevented :
    'returnValue' in source ? source.returnValue === false :
    source.getPreventDefault && source.getPreventDefault())
  event.isDefaultPrevented = returnTrue
  }
```

这是对浏览器 `preventDefault` 不同实现的兼容。

```javascript
source.defaultPrevented !== undefined ? source.defaultPrevented : '三元表达式'
```

如果浏览器支持 `defaultPrevented`， 则返回 `defaultPrevented` 的值

```javascript
'returnValue' in source ? source.returnValue === false : '后一个判断'
```

`returnValue` 默认为 `true`，如果阻止了浏览器的默认行为， `returnValue` 会变为 `false` 。

```javascript
source.getPreventDefault && source.getPreventDefault()
```

如果浏览器支持 `getPreventDefault` 方法，则调用 `getPreventDefault()` 方法获取是否阻止浏览器的默认行为。

判断为 `true` 的时候，将 `isDefaultPrevented` 设置为 `returnTrue` 方法。

### createProxy，创建代理对象

```javascript
ignoreProperties = /^([A-Z]|returnValue$|layer[XY]$|webkitMovement[XY]$)/,
function createProxy(event) {
  var key, proxy = { originalEvent: event }
  for (key in event)
    if (!ignoreProperties.test(key) && event[key] !== undefined) proxy[key] = event[key]

    return compatible(proxy, event)
}
```

`zepto` 中，事件触发的时候，返回给我们的 `event` 都不是原生的 `event` 对象，都是代理对象，这个就是代理对象的创建方法。

`ignoreProperties` 用来排除 `A-Z` 开头，即所有大写字母开头的属性，还有以`returnValue` 结尾，`layerX/layerY` ，`webkitMovementX/webkitMovementY` 结尾的非标准属性。

```javascript
for (key in event)
  if (!ignoreProperties.test(key) && event[key] !== undefined) proxy[key] = event[key]
```

遍历原生事件对象，排除掉不需要的属性和值为 `undefined` 的属性，将属性和值复制到代理对象上。

最终返回的是修正后的代理对象

### eventCapture

```javascript
function eventCapture(handler, captureSetting) {
  return handler.del &&
    (!focusinSupported && (handler.e in focus)) ||
    !!captureSetting
}
```

返回 `true` 表示在捕获阶段执行事件句柄，否则在冒泡阶段执行。

如果存在事件代理，并且事件为 `focus/blur` 事件，在浏览器不支持 `focusin/focusout` 事件时，设置为 `true` ， 在捕获阶段处理事件，间接达到冒泡的目的。

否则作用自定义的 `captureSetting` 设置事件执行的时机。

### add，Event 模块的核心方法

```javascript
function add(element, events, fn, data, selector, delegator, capture){
  var id = zid(element), set = (handlers[id] || (handlers[id] = []))
  events.split(/\s/).forEach(function(event){
    if (event == 'ready') return $(document).ready(fn)
    var handler   = parse(event)
    handler.fn    = fn
    handler.sel   = selector
    // emulate mouseenter, mouseleave
    if (handler.e in hover) fn = function(e){
      var related = e.relatedTarget
      if (!related || (related !== this && !$.contains(this, related)))
        return handler.fn.apply(this, arguments)
        }
    handler.del   = delegator
    var callback  = delegator || fn
    handler.proxy = function(e){
      e = compatible(e)
      if (e.isImmediatePropagationStopped()) return
      e.data = data
      var result = callback.apply(element, e._args == undefined ? [e] : [e].concat(e._args))
      if (result === false) e.preventDefault(), e.stopPropagation()
      return result
    }
    handler.i = set.length
    set.push(handler)
    if ('addEventListener' in element)
      element.addEventListener(realEvent(handler.e), handler.proxy, eventCapture(handler, capture))
      })
}
```

`add` 方法是向元素添加事件及事件响应，参数比较多，先来看看各参数的含义：

```javascript
element // 事件绑定的元素
events // 需要绑定的事件列表
fn // 事件执行时的句柄
data // 事件执行时，传递给事件对象的数据
selector // 事件绑定元素的选择器
delegator // 事件委托函数 
capture // 那个阶段执行事件句柄
```

```javascript
var id = zid(element), set = (handlers[id] || (handlers[id] = []))
```

获取或设置 `id` ， `set` 为事件句柄容器。

```javascript
events.split(/\s/).forEach(function(event){})
```

对每个事件进行处理

```javascript
if (event == 'ready') return $(document).ready(fn)
```

如果为 `ready` 事件，则调用 `ready` 方法，中止后续的执行

```javascript
var handler   = parse(event)
handler.fn    = fn
handler.sel   = selector
// emulate mouseenter, mouseleave
if (handler.e in hover) fn = function(e){
  var related = e.relatedTarget
  if (!related || (related !== this && !$.contains(this, related)))
    return handler.fn.apply(this, arguments)
    }
handler.del   = delegator
var callback  = delegator || fn
```

这段代码是设置 `handler` 上的一些属性，缓存起来。

这里主要看对 `mouseenter` 和 `mouseleave` 事件的模拟，具体的原理上面已经说过，只有在条件成立的时候才会执行事件句柄。

```javascript
handler.proxy = function(e){
  e = compatible(e)
  if (e.isImmediatePropagationStopped()) return
  e.data = data
  var result = callback.apply(element, e._args == undefined ? [e] : [e].concat(e._args))
  if (result === false) e.preventDefault(), e.stopPropagation()
  return result
}
```

事件句柄的代理函数。

`e` 为事件执行时的原生 `event` 对象，因此先调用 `compatible` 对 `e` 进行修正。

调用 `isImmediatePropagationStopped` 方法，看是否已经执行过 `stopImmediatePropagation` 方法，如果已经执行，则中止后续程序的执行。

再扩展 `e` 对象，将 `data` 存到 `e` 的 `data` 属性上。

执行事件句柄，将 `e` 对象作为句柄的第一个参数。

如果执行完毕后，显式返回 `false`，则阻止浏览器的默认行为和事件冒泡。

```javascript
set.push(handler)
if ('addEventListener' in element)
  element.addEventListener(realEvent(handler.e), handler.proxy, eventCapture(handler, capture))
```

将句柄存入句柄容器

调用元素的 `addEventListener` 方法，添加事件，事件的回调函数用的是句柄的代理函数，`eventCapture(handler, capture)` 来用指定是否在捕获阶段执行。

### remove，删除事件

```javascript
function remove(element, events, fn, selector, capture){
  var id = zid(element)
  ;(events || '').split(/\s/).forEach(function(event){
    findHandlers(element, event, fn, selector).forEach(function(handler){
      delete handlers[id][handler.i]
      if ('removeEventListener' in element)
        element.removeEventListener(realEvent(handler.e), handler.proxy, eventCapture(handler, capture))
        })
  })
}
```

首先获取指定元素的 `_zid`

```javascript
;(events || '').split(/\s/).forEach(function(event){})
```

遍历需要删除的 `events` 

```javascript
findHandlers(element, event, fn, selector).forEach(function(handler){})
```

调用 `findHandlers` 方法，查找 `event` 下需要删除的事件句柄

```javascript
delete handlers[id][handler.i]
```

删除句柄容器中对应的事件，在 `add` 函数中的句柄对象中的 `i` 属性就用在这里了，方便查找需要删除的句柄。

```javascript
element.removeEventListener(realEvent(handler.e), handler.proxy, eventCapture(handler, capture))
```

调用 `removeEventListener` 方法，删除对应的事件。

## 工具函数



## 方法



## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)



## 系列文章

1. [读Zepto源码之代码结构](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E4%BB%A3%E7%A0%81%E7%BB%93%E6%9E%84.md)
2. [读 Zepto 源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md)
3. [读Zepto源码之工具函数](https://github.com/yeyuqiudeng/reading-zepto/blob/a4d6ad99c57047beae2b652b4d2cbb380599a524/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md)
4. [读Zepto源码之神奇的$](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E7%A5%9E%E5%A5%87%E7%9A%84%24.md)
5. [读Zepto源码之集合操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E6%93%8D%E4%BD%9C.md)
6. [读Zepto源码之集合元素查找](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E5%85%83%E7%B4%A0%E6%9F%A5%E6%89%BE.md)
7. [读Zepto源码之操作DOM](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E6%93%8D%E4%BD%9CDOM.md)
8. [读Zepto源码之样式操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E6%A0%B7%E5%BC%8F%E6%93%8D%E4%BD%9C.md)
9. [读Zepto源码之属性操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B1%9E%E6%80%A7%E6%93%8D%E4%BD%9C.md)

## 参考

* [mouseenter与mouseover为何这般纠缠不清？](https://juejin.im/post/5935773fa0bb9f0058edbd61)
* [向zepto.js学习如何手动(trigger)触发DOM事件](https://juejin.im/post/5936f13b2f301e0058796482)
* [谁说你只是 "会用"jQuery?](https://juejin.im/post/5939956b5c497d006b690fee)
* [Zepto源码分析-event模块](http://www.cnblogs.com/mominger/p/4384692.html)
* [zepto源码之event.js](http://blog.csdn.net/u013055396/article/details/74907136)
* [说说focus /focusin /focusout /blur 事件](https://segmentfault.com/a/1190000003942014)
* [MDN:mouseenter](https://developer.mozilla.org/en-US/docs/Web/Events/mouseenter)
* [MDN:mouseleave](mouseleave)
* [MDN:MouseEvent.relatedTarget](https://developer.mozilla.org/en-US/docs/Web/API/MouseEvent/relatedTarget)
* [MDN:Event reference](https://developer.mozilla.org/en-US/docs/Web/Events)
* [MDN:Document.createEvent()](https://developer.mozilla.org/en-US/docs/Web/API/Document/createEvent)
* [MDN:event.stopImmediatePropagation](https://developer.mozilla.org/zh-CN/docs/Web/API/Event/stopImmediatePropagation)

## License

[![License: CC BY-NC-ND 4.0](https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://user-gold-cdn.xitu.io/2017/5/30/76626b0be42083d36b36f4a117dc1873) 

作者：对角另一面