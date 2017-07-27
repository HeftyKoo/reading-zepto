# 读Zepto源码之Deferred模块

`Deferred` 模块也不是必备的模块，但是 `ajax` 模块中，要用到 `promise` 风格，必需引入  `Deferred` 模块。`Deferred` 也用到了上一篇文章《[读Zepto源码之Callbacks模块]((https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BCallbacks%E6%A8%A1%E5%9D%97.md) )》介绍的 `Callbacks` 模块。

读 Zepto 源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## Promise/A+ 规范

规范的具体内容可以参与《[Promises/A+](https://promisesaplus.com/)》 和对应的中文翻译 《[Promise/A+规范](https://segmentfault.com/a/1190000002452115)》，这里只简单总结一下。

`promise` 是一个包含兼容 `promise` 规范的函数或对象，`promise` 包含三种状态 `pending` 进行中、`fulfilled` 已完成， `rejected` 被拒绝，并且必须处于其中一种状态。

`pending` 状态可以转换成 `fulfilled` 状态或者 `rejected` 状态，但是 `fulfilled` 状态和 `rejected` 状态不能再转换成其他状态。

`promise` 必须包含 `then` 方法，`then` 方法可以接收两个参数，参数类型都为函数，分别为状态变为 `fulfilled` 后调用的 `onFulfilled` 函数和 `rejected` 后调用的 `onRejected` 函数。

大致了解 `Promise/A+` 规范后，对后面源码的阅读会有帮助。

## Deferred 模块的整体结构

```javascript
;(function($){
  function Deferred(func) {
    deferred = {}
    if (func) func.call(deferred, deferred)
    return deferred
  }
  return $.Deferred = Deferred
})(Zepto)
```

从上面的精简的结构可以看出，`Deferred` 是一个函数，函数的返回值是一个符合 `Promise/A+` 规范的对象，如果 `Deferred` 有传递函数作为参数，则以 `deferred` 作为上下文，以 `deferred` 作为参数执行该函数。

## done、fail、progress、resolve/resolveWith、reject/rejectWith、notify/notifyWith 方法的生成

```javascript
var tuples = [
  // action, add listener, listener list, final state
  [ "resolve", "done", $.Callbacks({once:1, memory:1}), "resolved" ],
  [ "reject", "fail", $.Callbacks({once:1, memory:1}), "rejected" ],
  [ "notify", "progress", $.Callbacks({memory:1}) ]
],
    state = "pending",
    promise = {
      ...
    }
    deferred = {}
$.each(tuples, function(i, tuple){
  var list = tuple[2],
      stateString = tuple[3]

  promise[tuple[1]] = list.add

  if (stateString) {
    list.add(function(){
      state = stateString
    }, tuples[i^1][2].disable, tuples[2][2].lock)
  }

  deferred[tuple[0]] = function(){
    deferred[tuple[0] + "With"](this === deferred ? promise : this, arguments)
    return this
  }
  deferred[tuple[0] + "With"] = list.fireWith
})
```

### 变量解释

* tuples: 用来储存状态切换的方法名，对应状态的执行方法，回调关系列表和最终的状态描述。
* state: 状态描述
* promise：`promise` 包含执行方法 `always` 、`then` 、`done`、 `fail` 、`progress` 和辅助方法 `state` 、 `promise` 等
* deferred:  `deferred` 除了继承 `promise` 的方法外，还增加了切换方法， `resolve` 、`resoveWith` 、`reject` 、 `rejectWith` 、`notify` 、 `notifyWith` 。
  ​

### done、fail和progress的生成

```javascript
$.each(tuples, function(i, tuple){
  ...
})
```

方法的生成，通过遍历 `tuples` 实现

```javascript
var list = tuple[2],
	stateString = tuple[3]

promise[tuple[1]] = list.add
```

`list` 是工厂方法 `$.Callbacks` 生成的管理回调函数的一系列方法。具体参见上一篇文章《[读Zepto源码之Callbacks模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BCallbacks%E6%A8%A1%E5%9D%97.md)》。注意，`tuples` 的所有项中的 `$Callbacks` 都配置了 `memory:1` ，即开启记忆模式，增加的方法都会立即触发。包含 `resove` 和 `reject` 的项都传递了 `once:1` ，即回调列表只能触发一次。

`stateString` 是状态描述，只有包含了 `resolve` 和 `reject` 的数组项才具有。

`index` 为 `1` 的项，取出来的分别为 `done` 、 `fail` 和 `progress` ，所以 `promise` 上的 `done`、 `fail` 和 `progress` 方法，调用的是 `Callbacks` 中的 `add` 方法，实质是往各自的回调列表中添加回调函数。

### 状态切换

```javascript
if (stateString) {
  list.add(function(){
    state = stateString
  }, tuples[i^1][2].disable, tuples[2][2].lock)
}
```

如果 `stateString` 存在，即包含 `resolve` 和 `reject` 的数组项，则往对应的回调列表中添加切换 `state` 状态的方法，将 `state` 更改为对应方法触发后的状态。

同时，将状态锁定，即状态变为 `resolved` 或 `rejected` 状态后，不能再更改为其他状态。这里用来按位异或运算符 `^` 来实现。当 `i` 为 `0` ，即状态变为 `resolved` 时， `i^1` 为 `1` 。`tuples[i^1][2].disable` 将 `rejected` 的回调列表禁用，当 `i` 为 `1` 时， `i^1` 为 `0` ，将 `resolved` 的回调列表禁用。即实现了成功和失败的状态互斥，做得状态锁定，不能再更改。

在状态变更后，同时将 `tuples[2]` 的回调列表锁定，要注意 `disable` 和 `lock` 的区别，具体见《[读Zepto源码之Callbacks模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BCallbacks%E6%A8%A1%E5%9D%97.md#lock-和-disable-的区别)》，由于这里用了记忆模式，所以还可以往回调列表中添加回调方法，并且回调方法会立即触发。

### resolve/resolveWith、reject/rejectWith、notify/notifyWith 方法的生成

```javascript
deferred[tuple[0] + "With"] = list.fireWith

deferred[tuple[0]] = function(){
  deferred[tuple[0] + "With"](this === deferred ? promise : this, arguments)
  return this
}
```

这几个方法，存入在 `deferred` 对象中，并没有存入 `promise` 对象。

`resolveWith` 、 `rejectWith` 和 `notifyWith` 方法，其实等价于 `Callback` 的 `fireWith` 方法，`fireWith` 方法的第一个参数为上下文对象。

从源码中可以看到 `resolve` 、`reject` 和 `notify` 方法，调用的是对应的 `With` 后缀方法，如果当前上下文为 `deferred` 对象，则传入 `promise` 对象作为上下文。





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
10. [读Zepto源码之Event模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BEvent%E6%A8%A1%E5%9D%97.md)
11. [读Zepto源码之IE模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BIE%E6%A8%A1%E5%9D%97.md)
12. [读Zepto源码之Callbacks模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BCallbacks%E6%A8%A1%E5%9D%97.md)



## 参考

* [Zepto源码分析-deferred模块](http://www.cnblogs.com/mominger/p/4411632.html)
* [Promises/A+](https://promisesaplus.com/)
* [Promise/A+规范](https://segmentfault.com/a/1190000002452115)
* [jQuery的deferred对象详解](http://www.ruanyifeng.com/blog/2011/08/a_detailed_explanation_of_jquery_deferred_object.html)

## License

[![License: CC BY-NC-ND 4.0](https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://user-gold-cdn.xitu.io/2017/5/30/76626b0be42083d36b36f4a117dc1873) 

作者：对角另一面