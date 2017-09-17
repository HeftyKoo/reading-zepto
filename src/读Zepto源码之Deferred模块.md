# 读Zepto源码之Deferred模块

`Deferred` 模块也不是必备的模块，但是 `ajax` 模块中，要用到 `promise` 风格，必需引入  `Deferred` 模块。`Deferred` 也用到了上一篇文章《[读Zepto源码之Callbacks模块]((https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BCallbacks%E6%A8%A1%E5%9D%97.md) )》介绍的 `Callbacks` 模块。

读 Zepto 源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## GitBook

《[reading-zepto](https://yeyuqiudeng.gitbooks.io/reading-zepto/content/)》

## Promise/A+ 规范

规范的具体内容可以参考《[Promises/A+](https://promisesaplus.com/)》 和对应的中文翻译 《[Promise/A+规范](https://segmentfault.com/a/1190000002452115)》，这里只简单总结一下。

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

## promise 对象

### .state()

```javascript
state: function() {
  return state
},
```

`state` 方法的作用是返回当前的状态。

### .always()

```javascript
always: function() {
  deferred.done(arguments).fail(arguments)
  return this
},
```

`always` 是一种省事的写法，即无论成功还是失败，都会执行回调。调用的是 `deferred` 上的 `done` 和 `fail` 方法。或许你会有疑惑，`done` 和 `fail` 方法，上面的分析中，明明是 `promise` 的方法，为什么 `deferred` 对象上也有这两个方法呢，这个下面会讲到。

### .promise()

```javascript
promise: function(obj) {
  return obj != null ? $.extend( obj, promise ) : promise
}
```

返回 `promise` 对象，如果 `obj` 有传递，则将 `promise` 上的方法扩展到 `obj` 上。

### .then()

```javascript
then: function(/* fnDone [, fnFailed [, fnProgress]] */) {
  var fns = arguments
  return Deferred(function(defer){
    $.each(tuples, function(i, tuple){
      var fn = $.isFunction(fns[i]) && fns[i]
      deferred[tuple[1]](function(){
        var returned = fn && fn.apply(this, arguments)
        if (returned && $.isFunction(returned.promise)) {
          returned.promise()
            .done(defer.resolve)
            .fail(defer.reject)
            .progress(defer.notify)
        } else {
          var context = this === promise ? defer.promise() : this,
              values = fn ? [returned] : arguments
          defer[tuple[0] + "With"](context, values)
        }
      })
    })
    fns = null
  }).promise()
}
```

`promise` 的 `then` 方法接收三个参数，分别为成功的回调、失败的回调和进度的回调。

### then整体结构

将 `then` 简化后，可以看到以下的结构：

```javascript
return Deferred(function(defer){}).promise()
```

返回的是 `deferred` 对象，`deferred` 对象上的 `promise` 方法，其实就是 `promise` 对象上的 `promise` 方法，所以 `then` 方法，最终返回的还是 `promise` 对象。所以 `promise` 可以这样一直调用下去 `promise().then().then()....` 。

### Deferred 调用

```javascript
var fns = arguments
return Deferred(function(defer) {
  ...
})
fns = null
```

这里的变量 `fns` 是 `then` 所传入的参数，即上文提到的三个回调。

最后的 `fns = null` ，是释放引用，让 `JS` 引擎可以进行垃圾回收。

`Deferred` 的参数是一个函数，上文在分析总体结构的时候，有一句关键的代码 `if (func) func.call(deferred, deferred)` 。所以这里的函数的参数  `defer` 即为 `deferred` 对象。

### 执行回调

```javascript
$.each(tuples, function(i, tuple){
  var fn = $.isFunction(fns[i]) && fns[i]
  deferred[tuple[1]](function(){
    var returned = fn && fn.apply(this, arguments)
    if (returned && $.isFunction(returned.promise)) {
      returned.promise()
        .done(defer.resolve)
        .fail(defer.reject)
        .progress(defer.notify)
    } else {
      var context = this === promise ? defer.promise() : this,
          values = fn ? [returned] : arguments
      defer[tuple[0] + "With"](context, values)
    }
  })
})
```

遍历 `tuples` ， `tuples` 中的顺序，跟 `then` 中规定 `done` 、`fail` 和 `progress` 的回调顺序一致。

所以用 `var fn = $.isFunction(fns[i]) && fns[i]` 来判断对应位置的参数是否为 `function` 类型，如果是，则赋值给 `fn` 。

`deferred[tuple[1]]` 是对应的是 `done`、`fail` 和 `progress` 。所以在 `then` 里，会依次执行这三个方法。

```javascript
var returned = fn && fn.apply(this, arguments)
```

`returned` 是  `then` 中三个回调方法执行后返回的结果。

```javascript
if (returned && $.isFunction(returned.promise)) {
  returned.promise()
    .done(defer.resolve)
    .fail(defer.reject)
    .progress(defer.notify)
}
```

如果回调返回的是 `promise` 对象，调用新 `promise` 对象中的 `promise` 方法，新 `promise` 对象切换状态时， 并将当前 `deferred` 对象对应的状态切换方法传入，在新 `promise` 切换状态时执行。这就实现了两个 `promise` 对象的状态交流。

```javascript
var context = this === promise ? defer.promise() : this,
    values = fn ? [returned] : arguments
defer[tuple[0] + "With"](context, values)
```

如果返回的不是 `promise` 对象，则判断 `this` 是否为 `promise` ，如果是，则返回 `defer.promise()` ，修正执行的上下文。

然后调用对应的状态切换方法切换状态。

### promise 对象与 deferred 对象

```javascript
promise.promise(deferred)
```

从上面的分析中，可以看到，`deferred` 对象上并没有`done` 、 `fail` 和 `progress` 方法，这是从 `promise` 上扩展来的。

既然已经有了一个拥有 `promise` 对象的所有方法的 `deferred` 对象，为什么还要一个额外的 `promise` 对象呢？

`promise` 对象上没有状态切换方法，所以在 `then` 中，要绑定上下文的时候时候，绑定的都是 `promise` 对象，这是为了避免在执行的过程中，将执行状态改变。 

## $.when

```javascript
$.when = function(sub) {
    var resolveValues = slice.call(arguments),
        len = resolveValues.length,
        i = 0,
        remain = len !== 1 || (sub && $.isFunction(sub.promise)) ? len : 0,
        deferred = remain === 1 ? sub : Deferred(),
        progressValues, progressContexts, resolveContexts,
        updateFn = function(i, ctx, val){
          return function(value){
            ctx[i] = this
            val[i] = arguments.length > 1 ? slice.call(arguments) : value
            if (val === progressValues) {
              deferred.notifyWith(ctx, val)
            } else if (!(--remain)) {
              deferred.resolveWith(ctx, val)
            }
          }
        }

    if (len > 1) {
      progressValues = new Array(len)
      progressContexts = new Array(len)
      resolveContexts = new Array(len)
      for ( ; i < len; ++i ) {
        if (resolveValues[i] && $.isFunction(resolveValues[i].promise)) {
          resolveValues[i].promise()
            .done(updateFn(i, resolveContexts, resolveValues))
            .fail(deferred.reject)
            .progress(updateFn(i, progressContexts, progressValues))
        } else {
          --remain
        }
      }
    }
    if (!remain) deferred.resolveWith(resolveContexts, resolveValues)
    return deferred.promise()
  }
```

`when` 方法用来管理一系列的异步队列，如果所有的异步队列都执行成功，则执行成功方法，如果有一个异步执行失败，则执行失败方法。这个方法也可以传入非异步方法。

### 一些变量

```javascript
var resolveValues = slice.call(arguments),
    len = resolveValues.length,
    i = 0,
    remain = len !== 1 || (sub && $.isFunction(sub.promise)) ? len : 0,
    deferred = remain === 1 ? sub : Deferred(),
    progressValues, progressContexts, resolveContexts,
```

* resolveValues：所有的异步对象，用 `slice` 转换成数组形式。
* len: 异步对象的个数。
* remain: 剩余个数。这里还有个判断，是为了确定只有一个参数时，这个参数是不是异步对象，如果不是，则 `remain` 初始化为 `0` 。其他情况，初始化为当前的个数。
* i: 当前异步对象执行的索引值。
* deferred: `deferred` 对象，如果只有一个异步对象（只有一个参数，并且不为异步对象时， `remain` 为 `0` ），则直接使用当前的 `deferred` 对象，否则创建一个新的 `deferred` 对象。
* progressValues： 进度回调函数数组。
* progressContexts： 进度回调函数绑定的上下文数组
* resolveContexts： 成功回调函数绑定的上下文数组

### updateFn

```javascript
updateFn = function(i, ctx, val){
  return function(value){
    ctx[i] = this
    val[i] = arguments.length > 1 ? slice.call(arguments) : value
    if (val === progressValues) {
      deferred.notifyWith(ctx, val)
    } else if (!(--remain)) {
      deferred.resolveWith(ctx, val)
    }
  }
}
```

`updateFn` 方法，在每个异步对象执行 `resolve` 方法和 `progress` 方法时都调用。

参数 `i` 为异步对象的索引值，参数 `ctx`  为对应的上下文数组，即 `resolveContexts` 或 `resolveContexts` ， `val` 为对应的回调函数数组，即 `progresValues` 或 `resolveValues` 。

```javascript
if (val === progressValues) {
	deferred.notifyWith(ctx, val)
}
```

如果为 `progress` 的回调，则调用 `deferred` 的 `notifyWith` 方法。

```javascript
else if (!(--remain)) {
  deferred.resolveWith(ctx, val)
}
```

否则，将 `remain` 减少 `1`，如果回调已经执行完毕，则调用 `deferred` 的 `resolveWith` 方法。

### 依次处理异步对象

```javascript
if (len > 1) {
  progressValues = new Array(len)
  progressContexts = new Array(len)
  resolveContexts = new Array(len)
  for ( ; i < len; ++i ) {
    if (resolveValues[i] && $.isFunction(resolveValues[i].promise)) {
      resolveValues[i].promise()
        .done(updateFn(i, resolveContexts, resolveValues))
        .fail(deferred.reject)
        .progress(updateFn(i, progressContexts, progressValues))
    } else {
      --remain
    }
  }
}
```

首先初始化 `progressValues` 、`progressContexts` 和 `resolveContexts` ，数组长度为异步对象的长度。

```javascript
if (resolveValues[i] && $.isFunction(resolveValues[i].promise)) {
  resolveValues[i].promise()
    .done(updateFn(i, resolveContexts, resolveValues))
    .fail(deferred.reject)
    .progress(updateFn(i, progressContexts, progressValues))
}
```

如果为 `promise` 对象，则调用对应的 `promise` 方法。

```javascript
else {
  --remain
}
```

如果不是 `promise` 对象，则将 `remian` 减少 `1` 。

```javascript
if (!remain) deferred.resolveWith(resolveContexts, resolveValues)
return deferred.promise()
```

如果无参数，或者参数不是异步对象，或者所有的参数列表都不是异步对象，则直接调用 `resoveWith` 方法，调用成功函数列表。

最后返回的是 `promise` 对象。 

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

[署名-非商业性使用-禁止演绎 4.0 国际 (CC BY-NC-ND 4.0)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://user-gold-cdn.xitu.io/2017/5/30/76626b0be42083d36b36f4a117dc1873) 

作者：对角另一面