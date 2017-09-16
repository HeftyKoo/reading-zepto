# 读Zepto源码之代码结构

虽然最近工作中没有怎么用 zepto ，但是据说 zepto 的源码比较简单，而且网上的资料也比较多，所以我就挑了 zepto 下手，希望能为以后阅读其他框架的源码打下基础吧。

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

阅读zepto之前需要了解 javascript 原型链和闭包的知识，推荐阅读王福朋的这篇文章:[深入理解 Javascript 原型和闭包](http://www.cnblogs.com/wangfupeng1988/p/3977924.html)，写得很详细，也非常易于阅读。

## 源码结构

### 整体结构

```javascript
var Zepto = (function () {
  ...
})()

window.Zepto = Zepto
window.$ === undefined && (window.$ = Zepto)
```

如果在编辑器中将 zepto 的源码折叠起来，看到的就跟上面的代码一样。

zepto 的核心是一个闭包，加载完毕后立即执行。然后暴露给全局变量 `zepto` ，如果 `$` 没有定义，也将 `$` 赋值为 `Zepto` 。

## 核心结构

在这部分中，我们先不关注 zepto 的具体实现，只看核心的结构，因此我将zepto中的逻辑先移除，得出如下的核心结构：

```javascript
var zepto = {}, $

function Z(doms) {
  var len = doms.length 
  for (var i = 0; i < len; i++) {
    this[i] = doms[i]
  }
  this.length = doms.length
}

zepto.Z = function(doms) {
  return new Z(doms)
}

zepto.init = function(doms) {
  var doms = ['domObj1','domObj2','domObj3']
  return zepto.Z(doms)
}

$ = function() {
  return zepto.init()
}

$.fn = {
  constructor: zepto.Z,
  method: function() {
    return this
  }
}

zepto.Z.prototype = Z.prototype = $.fn

return $
```

在源码中，可以看出， `$` 其实是一个函数，同时在 `$` 身上又挂了很多属性和方法（这里体现在 `$.fn` 身上，其他的会在后续的文章中谈到）。

我们在使用 zepto 的时候，会用 `$` 去获取 `dom` ，并且在这些  `dom` 对象身上都有 zepto 定义的各种各样的操作方法。

从上面的伪代码中，可以看到，`$` 其实调用了 `zepto.init()` 方法，在 `init` 方法中，会获取到 `dom` 元素集合，然后将集合交由 `zepto.Z()` 方法处理，而 `zepto.Z` 方法返回的是函数 `Z` 的一个实例。

函数 `Z`  会将 `doms` 展开，变成实例的属性，`key` 为对应 `domObj` 的索引， 并且设置实例的 `length` 属性。

## zepto.Z.prototype = Z.prototype = $.fn

读到这里，你可能会有点疑惑，`$` 最终返回的是 `Z` 函数的实例，但是 `Z` 函数明明没有 `dom` 的操作方法啊，这些操作方法都定义在 ` $.fn` 身上，为什么 `$` 可以调用这些方法呢？

其实关键在于这句代码 `Z.prototype = $.fn` ，这句代码将 `Z` 的 `prototype` 指向 `$.fn` ，这样，`Z` 的实例就继承了 `$.fn` 的方法。

既然这样就已经让 `Z` 的实例继承了 `$.fn` 的方法，那 `zepto.Z.prototype = $.fn` 又是为什么呢？

如果我们再看源码，会发现有这样的一个方法：

```javascript
zepto.isZ = function(object) {
  return object instanceof zepto.Z
}
```

这个方法是用来判读一个对象是否为 zepto 对象，这是通过判断这个对象是否为 `zepto.Z` 的实例来完成的，因此需要将 `zepto.Z` 和 `Z` 的 `prototype` 指向同一个对象。 `isZ` 方法会在 `init` 中用到，后面也会介绍。

## 参考

* [zepto源码分析-代码结构](https://segmentfault.com/a/1190000007515865)
* [zepto对象思想与源码分析](http://www.kancloud.cn/wangfupeng/zepto-design-srouce)
* [zepto设计和源码分析](http://www.imooc.com/learn/745)
* [zepto源码中关于`zepto.Z.prototype = $.fn`的问题](https://segmentfault.com/q/1010000005782663)


## License

[署名-非商业性使用-禁止演绎 4.0 国际 (CC BY-NC-ND 4.0)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：

  ![](https://segmentfault.com/img/bVCJ55?w=430&h=430)