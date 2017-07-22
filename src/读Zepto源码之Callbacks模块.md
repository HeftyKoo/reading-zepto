# 读Zepto源码之Callbacks模块

Callbacks 模块并不是必备的模块，其作用是管理回调函数，为 Defferred 模块提供支持，Defferred 模块又为 Ajax 模块的 `promise` 风格提供支持，接下来很快就会分析到 Ajax模块，在此之前，先看 Callbacks 模块和 Defferred 模块的实现。

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## 整体结构

将 Callbacks 模块的代码精简后，得到的结构如下：

```javascript
;(function($){
  $.Callbacks = function(options) {
    ...
    Callbacks = {
      ...
    }
    return Callbacks
  }
})(Zepto)
```

其实就是向 `zepto` 对象上，添加了一个 `Callbacks` 函数，这个是一个工厂函数，调用这个函数返回的是一个对象，对象内部包含了一系列的方法。

`options` 参数为一个对象，在源码的内部，作者已经注释了各个键值的含义。

```javascript
// Option flags:
  //   - once: Callbacks fired at most one time.
  //   - memory: Remember the most recent context and arguments
  //   - stopOnFalse: Cease iterating over callback list
  //   - unique: Permit adding at most one instance of the same callback
once: 回调至多只能触发一次
memory: 记下最近一次触发的上下文及参数列表，再添加新回调的时候都立刻用这个上下文及参数立即执行
stopOnFalse： 如果队列中有回调返回 `false`，立即中止后续回调的执行
unique: 同一个回调只能添加一次
```

## 全局参数

```javascript
options = $.extend({}, options)

var memory, // Last fire value (for non-forgettable lists)
    fired,  // Flag to know if list was already fired
    firing, // Flag to know if list is currently firing
    firingStart, // First callback to fire (used internally by add and fireWith)
    firingLength, // End of the loop when firing
    firingIndex, // Index of currently firing callback (modified by remove if needed)
    list = [], // Actual callback list
    stack = !options.once && [], // Stack of fire calls for repeatable lists
```

* `options` : 构造函数的配置，默认为空对象
* `list` ： 回调函数列表
* `stack` ： 列表可以重复触发时，用来缓存触发过程中未执行的任务参数，如果列表只能触发一次，`stack` 永远为 `false`
* `memory` : 记忆模式，会记住上一次触发的上下文及参数
* `fired` ： 回调函数列表已经触发过
* `firing` :  回调函数列表正在触发
* `firingStart` :  回调任务的开始位置
* `firingIndex` : 当前回调任务的索引
* `firingLength`：回调任务的长度

## 基础用法

我用 `jQuery` 和 `Zepto` 的时间比较短，之前也没有直接用过 `Callbacks` 模块，单纯看代码可能不易理解它是怎样工作的，在分析之前，先看一下简单的 `API` 调用，可能会有助于理解。

```javascript
var callbacks = $.Callbacks({memory: true})
var a = function(a) {
  console.log('a ' + a)
}
var b = function(b) {
  console.log('b ' + b)
}
var c = function(c) {
  console.log('c ' + c)
}
callbacks.add(a).add(b).add(c)  // 向队列 list 中添加了三个回调
callbacks.remove(c) // 删除 c
callbacks.fire('fire') 
// 到这步输出了 `a fire` `b fire` 没有输出 `c fire`
callbacks.lock()
callbacks.fire('fire after lock')  // 到这步没有任何输出
// 继续向队列添加回调，注意 `Callbacks` 的参数为 `memory: true`
callbacks.add(function(d) {  
  console.log('after lock')
})
// 输出 `after lock`
callbacks.disable()
callbacks.add(function(e) {
  console.log('after disable')
}) 
// 没有任何输出
```

上面的例子只是简单的调用，也有了注释，下面开始分析 `API`

## 内部方法

### fire

```javascript
fire = function(data) {
  memory = options.memory && data
  fired = true
  firingIndex = firingStart || 0
  firingStart = 0
  firingLength = list.length
  firing = true
  for ( ; list && firingIndex < firingLength ; ++firingIndex ) {
    if (list[firingIndex].apply(data[0], data[1]) === false && options.stopOnFalse) {
      memory = false
      break
    }
  }
  firing = false
  if (list) {
    if (stack) stack.length && fire(stack.shift())
    else if (memory) list.length = 0
    else Callbacks.disable()
      }
}
```

`Callbacks` 模块只有一个内部方法 `fire` ，用来触发 `list` 中的回调执行，这个方法是 `Callbacks` 模块的核心。

#### 变量初始化

```javascript
memory = options.memory && data
fired = true
firingIndex = firingStart || 0
firingStart = 0
firingLength = list.length
firing = true
```

`fire` 只接收一个参数 `data` ，这个内部方法 `fire` 跟我们调用 `API` 所接收的参数不太一样，这个 `data` 是一个数组，数组里面只有两项，第一项是上下文对象，第二项是回调函数的参数数组。

如果 `options.memory` 为 `true` ，则将 `data`，也即上下文对象和参数保存下来。

 将 `list` 是否已经触发过的状态 `fired` 设置为 `true`。

将当前回调任务的索引值 `firingIndex` 指向回调任务的开始位置 `firingStart` 或者回调列表的开始位置。

将回调列表的开始位置 `firingStart` 设置为回调列表的开始位置。

将回调任务的长度 `firingLength` 设置为回调列表的长度。

将回调的开始状态 `firing` 设置为 `true`

#### 执行回调

```javascript
for ( ; list && firingIndex < firingLength ; ++firingIndex ) {
  if (list[firingIndex].apply(data[0], data[1]) === false && options.stopOnFalse) {
    memory = false
    break
  }
}
firing = false
```

 执行回调的整体逻辑是遍历回调列表，逐个执行回调。

循环的条件是，列表存在，并且当前回调任务的索引值 `firingIndex` 要比回调任务的长度要小，这个很容易理解，当前的索引值都超出了任务的长度，就找不到任务执行了。

`list[firingIndex].apply(data[0], data[1])` 就是从回调列表中找到对应的任务，绑定上下文对象，和传入对应的参数，执行任务。

如果回调执行后显式返回 `false`， 并且 `options.stopOnFalse` 设置为 `true` ，则中止后续任务的执行，并且清空 `memory` 的缓存。

回调任务执行完毕后，将 `firing` 设置为 `false`，表示当前没有正在执行的任务。

#### 检测未执行的回调及清理工作

```javascript
if (list) {
  if (stack) stack.length && fire(stack.shift())
  else if (memory) list.length = 0
  else Callbacks.disable()
}
```

列表任务执行完毕后，先检查 `stack` 中是否有没有执行的任务，如果有，则将任务参数取出，调用 `fire` 函数执行。反而会看到，`stack` 储存的任务是 `push` 进去的，用 `shift` 取出，表明任务执行的顺序是先进先出。

`memory` 存在，则清空回调列表，用 `list.length = 0` 是清空列表的一个方法。在全局参数中，可以看到， `stack` 为 `false` ，只有一种情况，就是 `options.once` 为 `true` 的时候，表示任务只能执行一次，所以要将列表清空。而 `memory` 为 `true` ，表示后面添加的任务还可以执行，所以还必须保持 `list` 容器的存在，以便后续任务的添加和执行。

其他情况直接调用 `Callbacks.disable()` 方法，禁用所有回调任务的添加和执行。

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



## 参考

* [Zepto源码分析-callbacks模块](http://www.cnblogs.com/mominger/p/4369469.html)
* [读jQuery之十九（多用途回调函数列表对象）](http://www.cnblogs.com/snandy/archive/2012/11/15/2770237.html)

## License

[![License: CC BY-NC-ND 4.0](https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://user-gold-cdn.xitu.io/2017/5/30/76626b0be42083d36b36f4a117dc1873) 

作者：对角另一面