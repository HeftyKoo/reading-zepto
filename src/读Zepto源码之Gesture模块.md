# 读Zepto源码之Gesture模块

`Gesture` 模块基于 `IOS` 上的 `Gesture` 事件的封装，利用 `scale` 属性，封装出 `pinch` 系列事件。

读 Zepto 源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## 整体结构

```javascript
;(function($){
  if ($.os.ios) {
    var gesture = {}, gestureTimeout

    $(document).bind('gesturestart', function(e){
     ...
    }).bind('gesturechange', function(e){
      ...
    }).bind('gestureend', function(e){
     ...
    })

    ;['pinch', 'pinchIn', 'pinchOut'].forEach(function(m){
      $.fn[m] = function(callback){ return this.bind(m, callback) }
    })
  }
})(Zepto)

```

注意这里有个判断 `$.os.ios` ，用来判断是否为 `ios` 。这个判断需要引入设备侦测模块 `Detect` 。这个模块利用 `userAgent` 来进行设备侦测，里面是一大堆正则表达式，所以这个模块后面是不打算分析的了。

然后是监测 `gesturestart` 、`gesturechange`、 `gestureend` 事件，根据这三个事件，可以组合出 `pinch` 、`pinchIn` 和 `pinchOut` 事件。其实就是缩小和放大的手势操作。

其中变量 `gesture` 对象和 `Touch` 模块中的 `touch` 对象的作用差不多，可以先看看 《[读Zepto源码之Touch模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BTouch%E6%A8%A1%E5%9D%97.md)》对 `Touch` 模块的分析。

## parentIfText

```javascript
function parentIfText(node){
  return 'tagName' in node ? node : node.parentNode
}
```
这个辅助方法是获取目标节点，如果节点不是元素节点，则用父节点作为目标节点。如果事件在文本节点或者伪类元素上触发时，会出现不是元素节点的情况。

## 事件

### gesturestart

```javascript
bind('gesturestart', function(e){
  var now = Date.now(), delta = now - (gesture.last || now)
  gesture.target = parentIfText(e.target)
  gestureTimeout && clearTimeout(gestureTimeout)
  gesture.e1 = e.scale
  gesture.last = now
})
```

如 `Touch` 模块一样，在 `gesturestart` 时，也用 `delta` 来记录两次 `start` 之间的时间间隔，用 `gesture.target` 来保存目标元素，`e1` 是起点时的缩放值。

### gesturechange

```javascript
bind('gesturechange', function(e){
  gesture.e2 = e.scale
})
```

在 `gesturechange` 时，更新终点 `guesture.e2` 的缩放值。

### gestureend

```javascript
if (gesture.e2 > 0) {
  Math.abs(gesture.e1 - gesture.e2) != 0 && $(gesture.target).trigger('pinch') &&
    $(gesture.target).trigger('pinch' + (gesture.e1 - gesture.e2 > 0 ? 'In' : 'Out'))
  gesture.e1 = gesture.e2 = gesture.last = 0
} else if ('last' in gesture) {
  gesture = {}
}
```

如果 `gesture.e2` 存在（不可能有小于 `0` 的情况吧？），在起点的缩放值和终点的缩放值不相同的情况下，触发 `pinch` 事件；如果起点的缩放值比终点的缩放值大，则继续触发 `pinchIn` 事件，则缩小效果；如果起点的缩放值比终点的缩放值小，则继续触发 `pinchOut` 事件，即放大效果。

最终将 `e1` 、 `e2`  和 `last` 都设置为 `0` 。

在 `last` 不存在的情况下（在调用 `preventDefault` 时），将 `gesture` 清空。

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



## 参考

* [指尖下的js —— 多触式web前端开发之三：处理复杂手势](http://www.cnblogs.com/pifoo/archive/2011/05/22/webkit-touch-event-3.html)

## License

[署名-非商业性使用-禁止演绎 4.0 国际 (CC BY-NC-ND 4.0)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://user-gold-cdn.xitu.io/2017/5/30/76626b0be42083d36b36f4a117dc1873) 

作者：对角另一面