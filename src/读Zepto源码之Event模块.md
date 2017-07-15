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

在 `Event` 模块中，主要做了如下几件事：

* 提供简洁的API
* 统一不同浏览器的 `event` 对象
* 事件句柄缓存池，方便手动触发事件和解绑事件。
* 事件委托

## 内部方法



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

## License

[![License: CC BY-NC-ND 4.0](https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://user-gold-cdn.xitu.io/2017/5/30/76626b0be42083d36b36f4a117dc1873) 

作者：对角另一面