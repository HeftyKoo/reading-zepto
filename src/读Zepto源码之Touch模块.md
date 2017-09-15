# 读Zepto源码之Touch模块

大家都知道，因为历史原因，移动端上的点击事件会有 `300ms` 左右的延迟，`Zepto` 的 `touch` 模块解决的就是移动端点击延迟的问题，同时也提供了滑动的 `swipe` 事件。

读 Zepto 源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## 实现的事件

```javascript
;['swipe', 'swipeLeft', 'swipeRight', 'swipeUp', 'swipeDown',
  'doubleTap', 'tap', 'singleTap', 'longTap'].forEach(function(eventName){
  $.fn[eventName] = function(callback){ return this.on(eventName, callback) }
})
```

从上面的代码中可以看到，`Zepto` 实现了以下的事件：

* swipe: 滑动事件
* swipeLeft: 向左滑动事件
* swipeRight: 向右滑动事件
* swipeUp: 向上滑动事件
* swipeDown: 向下滑动事件
* doubleTap: 屏幕双击事件
* tap: 屏幕点击事件，比 `click` 事件响应更快
* singleTap: 屏幕单击事件
* longTap: 长按事件

并且为每个事件都注册了快捷方法。

## 内部方法

### swipeDirection

```javascript
function swipeDirection(x1, x2, y1, y2) {
  return Math.abs(x1 - x2) >=
    Math.abs(y1 - y2) ? (x1 - x2 > 0 ? 'Left' : 'Right') : (y1 - y2 > 0 ? 'Up' : 'Down')
}
```

返回的是滑动的方法。

`x1` 为 `x轴` 起点坐标， `x2` 为 `x轴` 终点坐标， `y1` 为 `y轴` 起点坐标， `y2` 为 `y轴` 终点坐标。

这里有多组三元表达式，首先对比的是 `x轴` 和 `y轴` 上的滑动距离，如果 `x轴` 的滑动距离比 `y轴` 大，则为左右滑动，否则为上下滑动。

在 `x轴` 上，如果起点位置比终点位置大，则为向左滑动，返回 `Left` ，否则为向右滑动，返回 `Right` 。

在 `y轴` 上，如果起点位置比终点位置大，则为向上滑动，返回 `Up` ，否则为向下滑动，返回 `Down` 。

### longTap

```javascript
var touch = {},
    touchTimeout, tapTimeout, swipeTimeout, longTapTimeout,
    longTapDelay = 750,
    gesture
function longTap() {
  longTapTimeout = null
  if (touch.last) {
    touch.el.trigger('longTap')
    touch = {}
  }
}
```

触发长按事件。

`touch` 对象保存的是触摸过程中的信息。

在触发 `longTap` 事件前，先将保存定时器的变量 `longTapTimeout` 释放，如果 `touch` 对象中存在 `last` ，则触发 `longTap` 事件， `last` 保存的是最后触摸的时间。最后将 `touch` 重置为空对象，以便下一次使用。

### cancelLongTap

```javascript
function cancelLongTap() {
  if (longTapTimeout) clearTimeout(longTapTimeout)
  longTapTimeout = null
}
```

撤销 `longTap` 事件的触发。

如果有触发 `longTap` 的定时器，清除定时器即可阻止 `longTap` 事件的触发。

最后同样需要将 `longTapTimeout` 变量置为 `null` ，等待垃圾回收。

### cancelAll

```javascript
function cancelAll() {
  if (touchTimeout) clearTimeout(touchTimeout)
  if (tapTimeout) clearTimeout(tapTimeout)
  if (swipeTimeout) clearTimeout(swipeTimeout)
  if (longTapTimeout) clearTimeout(longTapTimeout)
  touchTimeout = tapTimeout = swipeTimeout = longTapTimeout = null
  touch = {}
}
```

 清除所有事件的执行。

其实就是清除所有相关的定时器，最后将 `touch` 对象设置为 `null` 。

### isPrimaryTouch

```javascript
function isPrimaryTouch(event){
  return (event.pointerType == 'touch' ||
          event.pointerType == event.MSPOINTER_TYPE_TOUCH)
  && event.isPrimary
}
```

是否为主触点。

当 `pointerType` 为 `touch` 并且 `isPrimary` 为 `true` 时，才为主触点。 `pointerType` 可为 `touch` 、 `pen` 和 `mouse` ，这里只处理手指触摸的情况。

### isPointerEventType

```javascript
function isPointerEventType(e, type){
  return (e.type == 'pointer'+type ||
          e.type.toLowerCase() == 'mspointer'+type)
}
```

触发的是否为 `pointerEvent` 。

在低版本的移动端 IE 浏览器中，只实现了 `PointerEvent` ，并没有实现 `TouchEvent` ，所以需要这个来判断。

## 事件触发

### 整体分析

```javascript
$(document).ready(function(){
    var now, delta, deltaX = 0, deltaY = 0, firstTouch, _isPointerType

    $(document)
      .bind('MSGestureEnd', function(e){
        ...
      })
      .on('touchstart MSPointerDown pointerdown', function(e){
        ...
      })
      .on('touchmove MSPointerMove pointermove', function(e){
        ...
      })
      .on('touchend MSPointerUp pointerup', function(e){
        ...
      })
      
      .on('touchcancel MSPointerCancel pointercancel', cancelAll)

    $(window).on('scroll', cancelAll)
```

先来说明几个变量，`now` 用来保存当前时间， `delta` 用来保存两次触摸之间的时间差， `deltaX` 用来保存 `x轴` 上的位移， `deltaY` 来用保存 `y轴` 上的位移， `firstTouch` 保存初始触摸点的信息， `_isPointerType` 保存是否为 `pointerEvent` 的判断结果。

从上面可以看到， `Zepto` 所触发的事件，是从 `touch` 、 `pointer` 或者 IE 的 `guesture` 事件中，根据不同情况来计算出来的。这些事件都绑定在 `document` 上。

### IE Gesture 事件的处理

`IE` 的手势使用，需要经历三步：

1. 创建手势对象
2. 指定目标元素
3. 指定手势识别时需要处理的指针

```javascript
if ('MSGesture' in window) {
  gesture = new MSGesture()
  gesture.target = document.body
}
```

这段代码包含了前两步。

```javascript
on('touchstart MSPointerDown pointerdown', function(e){
  ...
  if (gesture && _isPointerType) gesture.addPointer(e.pointerId)
}
```

这段是第三步，用 `addPointer` 的方法，指定需要处理的指针。

```javascript
bind('MSGestureEnd', function(e){
  var swipeDirectionFromVelocity =
      e.velocityX > 1 ? 'Right' : e.velocityX < -1 ? 'Left' : e.velocityY > 1 ? 'Down' : e.velocityY < -1 ? 'Up' : null
  if (swipeDirectionFromVelocity) {
    touch.el.trigger('swipe')
    touch.el.trigger('swipe'+ swipeDirectionFromVelocity)
  }
})
```

接下来就是分析手势了，`Gesture` 里只处理 `swipe` 事件。

`velocityX` 和 `velocityY` 分别为 `x轴` 和 `y轴` 上的速率。这里以 `1`  或 `-1` 为临界点，判断 `swipe` 的方法。

如果 `swipe` 的方向存在，则触发 `swipe` 事件，同时也触发带方法的 `swipe` 事件。

### start

```javascript
on('touchstart MSPointerDown pointerdown', function(e){
  if((_isPointerType = isPointerEventType(e, 'down')) &&
     !isPrimaryTouch(e)) return
  firstTouch = _isPointerType ? e : e.touches[0]
  if (e.touches && e.touches.length === 1 && touch.x2) {
    touch.x2 = undefined
    touch.y2 = undefined
  }
  now = Date.now()
  delta = now - (touch.last || now)
  touch.el = $('tagName' in firstTouch.target ?
               firstTouch.target : firstTouch.target.parentNode)
  touchTimeout && clearTimeout(touchTimeout)
  touch.x1 = firstTouch.pageX
  touch.y1 = firstTouch.pageY
  if (delta > 0 && delta <= 250) touch.isDoubleTap = true
  touch.last = now
  longTapTimeout = setTimeout(longTap, longTapDelay)
  if (gesture && _isPointerType) gesture.addPointer(e.pointerId)
})
```

#### 过滤掉非触屏事件

```javascript
if((_isPointerType = isPointerEventType(e, 'down')) &&
   !isPrimaryTouch(e)) return
firstTouch = _isPointerType ? e : e.touches[0]
```

这里还将 `isPointerEventType` 的判断结果保存到了 `_isPointerType` 中，用来判断是否为 `PointerEvent` 。

这里的判断其实就是只接近 `PointerEvent` 和 `TouchEvent` ，并且 `TouchEvent` 的 `isPrimary` 必须为 `true` 。

因为 `TouchEvent` 支持多点触碰，这里只取触碰的第一点存入 `firstTouch` 变量。

#### 重置终点坐标

```javascript
if (e.touches && e.touches.length === 1 && touch.x2) {
  touch.x2 = undefined
  touch.y2 = undefined
}
```

如果还需要记录，终点坐标是需要更新的。

正常情况下，`touch` 对象会在 `touchEnd` 或者 `cancel` 的时候清空，但是如果用户自己调用了 `preventDefault` 等，就可能会出现没有清空的情况。

这里有一点不太明白，为什么只会在 `touches` 单点操作的时候才清空呢？多个触碰点的时候不需要清空吗？

#### 记录触碰点的信息

```javascript
now = Date.now()
delta = now - (touch.last || now)
touch.el = $('tagName' in firstTouch.target ?
             firstTouch.target : firstTouch.target.parentNode)
touchTimeout && clearTimeout(touchTimeout)
touch.x1 = firstTouch.pageX
touch.y1 = firstTouch.pageY
```

`now` 用来保存当前时间。

`delta` 用来保存两次点击时的时间间隔，用来处理双击事件。

`touch.el` 用来保存目标元素，这里有个判断，如果 `target` 不是标签节点时，取父节点作为目标元素。这会在点击伪类元素时出现。

如果 `touchTimeout` 存在，则清除定时器，避免重复触发。

`touch.x1` 和 `touch.y1` 分别保存 `x轴` 坐标和 `y轴` 坐标。

#### 双击事件

```javascript
if (delta > 0 && delta <= 250) touch.isDoubleTap = true
```

可以很清楚地看到， `Zepto` 将两次点击之间的时间间隔小于 `250ms` 时，作为 `doubleTap` 事件处理，将 `isDoubleTap` 设置为 `true` 。

#### 长按事件

```javascript
touch.last = now
longTapTimeout = setTimeout(longTap, longTapDelay)
```

将 `touch.last` 设置为当前时间。这样就可以记录两次点击时的时间差了。

同时开始长按事件定时器，从上面的代码可以看到，长按事件会在 `750ms` 后触发。

### move

```javascript
on('touchmove MSPointerMove pointermove', function(e){
  if((_isPointerType = isPointerEventType(e, 'move')) &&
     !isPrimaryTouch(e)) return
  firstTouch = _isPointerType ? e : e.touches[0]
  cancelLongTap()
  touch.x2 = firstTouch.pageX
  touch.y2 = firstTouch.pageY

  deltaX += Math.abs(touch.x1 - touch.x2)
  deltaY += Math.abs(touch.y1 - touch.y2)
})
```

`move` 事件处理了两件事，一是记录终点坐标，一是计算起点到终点之间的位移。

要注意这里还调用了 `cancelLongTap` 清除了长按定时器，避免长按事件的触发。因为有移动，肯定就不是长按了。

### end

```javascript
on('touchend MSPointerUp pointerup', function(e){
  if((_isPointerType = isPointerEventType(e, 'up')) &&
     !isPrimaryTouch(e)) return
  cancelLongTap()

  if ((touch.x2 && Math.abs(touch.x1 - touch.x2) > 30) ||
      (touch.y2 && Math.abs(touch.y1 - touch.y2) > 30))

    swipeTimeout = setTimeout(function() {
      if (touch.el){
        touch.el.trigger('swipe')
        touch.el.trigger('swipe' + (swipeDirection(touch.x1, touch.x2, touch.y1, touch.y2)))
      }
      touch = {}
    }, 0)

  else if ('last' in touch)
  
    if (deltaX < 30 && deltaY < 30) {
    
      tapTimeout = setTimeout(function() {
        
        var event = $.Event('tap')
        event.cancelTouch = cancelAll
        
        if (touch.el) touch.el.trigger(event)

        if (touch.isDoubleTap) {
          if (touch.el) touch.el.trigger('doubleTap')
          touch = {}
        }

        else {
          touchTimeout = setTimeout(function(){
            touchTimeout = null
            if (touch.el) touch.el.trigger('singleTap')
            touch = {}
          }, 250)
        }
      }, 0)
    } else {
      touch = {}
    }
  deltaX = deltaY = 0

})
```

#### swipe

```javascript
cancelLongTap()
if ((touch.x2 && Math.abs(touch.x1 - touch.x2) > 30) ||
    (touch.y2 && Math.abs(touch.y1 - touch.y2) > 30))

  swipeTimeout = setTimeout(function() {
    if (touch.el){
      touch.el.trigger('swipe')
      touch.el.trigger('swipe' + (swipeDirection(touch.x1, touch.x2, touch.y1, touch.y2)))
    }
    touch = {}
  }, 0)
```

进入 `end` 时，立刻清除 `longTap` 定时器的执行。

可以看到，起点和终点的距离超过 `30` 时，会被判定为 `swipe` 滑动事件。

在触发完 `swipe` 事件后，立即触发对应方向上的 `swipe` 事件。

注意，`swipe` 事件并不是在 `end` 系列事件触发时立即触发的，而是设置了一个 `0ms` 的定时器，让事件异步触发，这个有什么用呢？后面会讲到。

#### tap

```javascript
else if ('last' in touch)
  
  if (deltaX < 30 && deltaY < 30) {

    tapTimeout = setTimeout(function() {

      var event = $.Event('tap')
      event.cancelTouch = cancelAll

      if (touch.el) touch.el.trigger(event)

    }, 0)
  } else {
    touch = {}
  }
deltaX = deltaY = 0
```

终于看到重点了，首先判断 `last` 是否存在，从 `start` 中可以看到，如果触发了 `start` ， `last` 肯定是存在的，但是如果触发了长按事件，`touch` 对象会被清空，这时不会再触发 `tap` 事件。

如果不是 `swipe` 事件，也不存在 `last` ，则只将 `touch` 清空，不触发任何事件。

在最后会将 `deltaX` 和 `deltaY` 重置为 `0` 。

触发 `tap` 事件时，会在 `event` 中加了 `cancelTouch` 方法，外界可以通过这个方法取消所有事件的执行。

这里同样用了 `setTimeout` 异步触发事件。

#### doubleTap

```javascript
if (touch.isDoubleTap) {
  if (touch.el) touch.el.trigger('doubleTap')
  touch = {}
}
```

这个 `isDoubleTap` 在 `start` 时确定的，上面已经分析过了，在 `end` 的时候触发 `doubleTap` 事件。

因此，可以知道，在触发 `doubleTap` 事件之前会触发两次 `tap` 事件。



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



## 参考

* [zepto touch 库源码分析](https://segmentfault.com/a/1190000005882908)
* [PointerEvent](https://developer.mozilla.org/en-US/docs/Web/API/PointerEvent)
* [Pointer events](https://developer.mozilla.org/en-US/docs/Web/API/Pointer_events)
* [TouchEvent](https://developer.mozilla.org/en-US/docs/Web/API/TouchEvent)
* [Touch](https://developer.mozilla.org/en-US/docs/Web/API/Touch)
* [GestureEvent](https://developer.mozilla.org/en-US/docs/Web/API/GestureEvent)
* [MSGestureEvent](https://developer.mozilla.org/en-US/docs/Web/API/MSGestureEvent)
* [一步一步DIY zepto库，研究zepto源码8--touch模块](https://zrysmt.github.io/2017/04/28/%E4%B8%80%E6%AD%A5%E4%B8%80%E6%AD%A5DIY%20zepto%E5%BA%93%EF%BC%8C%E7%A0%94%E7%A9%B6zepto%E6%BA%90%E7%A0%818--touch%E6%A8%A1%E5%9D%97/)
* [zepto源码学习-06 touch](https://www.bbsmax.com/A/Vx5M9nPv5N/)
* [zepto源码之touch.js](http://blog.h5min.cn/u013055396/article/details/76606048)
* [addPointer method](https://msdn.microsoft.com/en-us/library/hh968251(v=vs.85).aspx)

## License

[![License: CC BY-NC-ND 4.0](https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://user-gold-cdn.xitu.io/2017/5/30/76626b0be42083d36b36f4a117dc1873) 

作者：对角另一面