# 读Zepto源码之fx_methods模块

`fx` 模块提供了 `animate` 动画方法，`fx_methods` 利用 `animate` 方法，提供一些常用的动画方法。所以 `fx_methods` 模块依赖于 `fx` 模块，在引入 `fx_methods` 前必须引入 `fx` 模块。

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## GitBook

《[reading-zepto](https://yeyuqiudeng.gitbooks.io/reading-zepto/content/)》

## 内部方法

### anim

```javascript
function anim(el, speed, opacity, scale, callback) {
  if (typeof speed == 'function' && !callback) callback = speed, speed = undefined
  var props = { opacity: opacity }
  if (scale) {
    props.scale = scale
    el.css($.fx.cssPrefix + 'transform-origin', '0 0')
  }
  return el.animate(props, speed, null, callback)
}
```

如果 `speed` 的参数类型为函数，并且 `callback` 没有传递，则认为 `speed` 位置的参数为 `callback`。

`props` 是过渡的属性， `fx_fethods` 主要实现 `show` 、 `hide` 和 `fadeIn`、  `fadeOut` 等动画，用到的过渡属性为 `opecity` 和 `scale` 。

当为 `scale` 时，将转换的原点设置为 `0 0`。

 最后调用的是 `fx` 模块中的 `animate` 方法。

### hide

```javascript
var document = window.document, docElem = document.documentElement,
    origShow = $.fn.show, origHide = $.fn.hide, origToggle = $.fn.toggle
function hide(el, speed, scale, callback) {
  return anim(el, speed, 0, scale, function(){
    origHide.call($(this))
    callback && callback.call(this)
  })
}
```

`hide` 方法其实就是将 `opacity` 的属性设置为 `0` 。在动画完成后，调用 `origHide` 方法，即原有的 `hide` 方法，将元素的 `display` 设置为 `none`。原有的 `hide` 方法分析见《[读Zepto源码之样式操作](https://github.com/yeyuqiudeng/reading-zepto/blob/6fb60c6a6ca1cf4f6846c32883774b5ba0f7de45/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E6%A0%B7%E5%BC%8F%E6%93%8D%E4%BD%9C.md#hide)》

## .show()

```javascript
$.fn.show = function(speed, callback) {
  origShow.call(this)
  if (speed === undefined) speed = 0
  else this.css('opacity', 0)
  return anim(this, speed, 1, '1,1', callback)
}
```

`show` 方法首先调用原有的 `hide` 方法，将元素显示出来，这是实现动画的基本条件。

如果没有设置 `speed`， 表示不需要动画，则过渡时间 `speed` 设置为 `0`。立即显示元素。

否则，先将 `opactity` 设置为 `0`， 再调用 `anim` 方法执行动画。`opacity` 设置为 `0` 也是执行动画的关键，从 `0` 变为 `1` 才有过渡的效果。

## .hide()

```javascript
$.fn.hide = function(speed, callback) {
  if (speed === undefined) return origHide.call(this)
  else return hide(this, speed, '0,0', callback)
}
```

如果 `speed` 没有传递，简单调用原有的 `hide` 方法即可，因为不需要过渡效果。

否则调用内部方法 `hide`。

## .toggle()

```javascript
$.fn.toggle = function(speed, callback) {
  if (speed === undefined || typeof speed == 'boolean')
    return origToggle.call(this, speed)
  else return this.each(function(){
    var el = $(this)
    el[el.css('display') == 'none' ? 'show' : 'hide'](speed, callback)
  })
}
```

`toggle` 方法是 `show` 和 `hide` 方法的切换。

如果 `speed` 没有传递，或者为 `boolean` 值，则表示不需要动画，调用原有的 `toggle` 方法即可。为什么要有一个 `boolean` 值的判断呢，这要看回 《[读Zepto源码之样式操作](https://github.com/yeyuqiudeng/reading-zepto/blob/6fb60c6a6ca1cf4f6846c32883774b5ba0f7de45/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E6%A0%B7%E5%BC%8F%E6%93%8D%E4%BD%9C.md#toggle)》关于 `toggle` 方法的分析了，原有的 `toggle` 方法接收一个参数，如果为 `true`，则指定调用 `show` 方法，否则调用 `hide` 方法。

否则，判断每个元素的 `display` 属性值，如果为 `none`，则调用 `show` 方法显示，否则调用 `hide` 方法隐藏。

 

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
20. [读Zepto源码之Fx模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BFx%E6%A8%A1%E5%9D%97.md)

## 参考

## License

[署名-非商业性使用-禁止演绎 4.0 国际 (CC BY-NC-ND 4.0)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://raw.githubusercontent.com/yeyuqiudeng/resource/master/images/qrcode_front-end-article.jpg) 

作者：对角另一面