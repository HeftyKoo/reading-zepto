# 读Zepto源码之Callbacks模块

Callbacks 模块并不是必备的模块，其作用是管理回调函数，为 Defferred 模块提供支持，Defferred 模块又为 Ajax 模块的 `promise` 风格提供支持，接下来很快就会分析到 Ajax模块，在此之前，先看 Callbacks 模块和 Defferred 模块的实现。

读 Zepto 源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

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

## 全局变量

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
* `memory` : 记忆模式下，会记住上一次触发的上下文及参数
* `fired` ： 回调函数列表已经触发过
* `firing` :  回调函数列表正在触发
* `firingStart` :  回调任务的开始位置
* `firingIndex` : 当前回调任务的索引
* `firingLength`：回调任务的长度

## 基础用法

我用 `jQuery` 和 `Zepto` 的时间比较短，之前也没有直接用过 `Callbacks` 模块，单纯看代码不易理解它是怎样工作的，在分析之前，先看一下简单的 `API` 调用，可能会有助于理解。

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

列表任务执行完毕后，先检查 `stack` 中是否有没有执行的任务，如果有，则将任务参数取出，调用 `fire` 函数执行。后面会看到，`stack` 储存的任务是 `push` 进去的，用 `shift` 取出，表明任务执行的顺序是先进先出。

`memory` 存在，则清空回调列表，用 `list.length = 0` 是清空列表的一个方法。在全局参数中，可以看到， `stack` 为 `false` ，只有一种情况，就是 `options.once` 为 `true` 的时候，表示任务只能执行一次，所以要将列表清空。而 `memory` 为 `true` ，表示后面添加的任务还可以执行，所以还必须保持 `list` 容器的存在，以便后续任务的添加和执行。

其他情况直接调用 `Callbacks.disable()` 方法，禁用所有回调任务的添加和执行。

## .add()

```javascript
add: function() {
  if (list) {
    var start = list.length,
        add = function(args) {
          $.each(args, function(_, arg){
            if (typeof arg === "function") {
              if (!options.unique || !Callbacks.has(arg)) list.push(arg)
                }
            else if (arg && arg.length && typeof arg !== 'string') add(arg)
              })
        }
    add(arguments)
    if (firing) firingLength = list.length
    else if (memory) {
      firingStart = start
      fire(memory)
    }
  }
  return this
},
```

`start` 为原来回调列表的长度。保存起来，是为了后面修正回调任务的开始位置时用。

### 内部方法add

```javascript
add = function(args) {
  $.each(args, function(_, arg){
    if (typeof arg === "function") {
      if (!options.unique || !Callbacks.has(arg)) list.push(arg)
        }
    else if (arg && arg.length && typeof arg !== 'string') add(arg)
      })
}
```

`add` 方法的作用是将回调函数 `push` 进回调列表中。参数 `arguments` 为数组或者伪数组。

用 `$.each` 方法来遍历 `args` ，得到数组项 `arg`，如果 `arg` 为 `function` 类型，则进行下一个判断。

在下一个判断中，如果 `options.unique` 不为 `true` ，即允许重复的回调函数，或者原来的列表中不存在该回调函数，则将回调函数存入回调列表中。

如果 `arg` 为数组或伪数组（通过 `arg.length` 是否存在判断，并且排除掉 `string` 的情况），再次调用 `add` 函数分解。

### 修正回调任务控制变量

```javascript
add(arguments)
if (firing) firingLength = list.length
else if (memory) {
  firingStart = start
  fire(memory)
}
```

调用 `add` 方法，向列表中添加回调函数。

如果回调任务正在执行中，则修正回调任务的长度 `firingLength` 为当前任务列表的长度，以便后续添加的回调函数可以执行。

否则，如果为 `memory` 模式，则将执行回调任务的开始位置设置为 `start` ，即原来列表的最后一位的下一位，也就是新添加进列表的第一位，然后调用 `fire` ，以缓存的上下文及参数 `memory` 作为 `fire` 的参数，立即执行新添加的回调函数。

## .remove()

```javascript
remove: function() {
  if (list) {
    $.each(arguments, function(_, arg){
      var index
      while ((index = $.inArray(arg, list, index)) > -1) {
        list.splice(index, 1)
        // Handle firing indexes
        if (firing) {
          if (index <= firingLength) --firingLength
          if (index <= firingIndex) --firingIndex
            }
      }
    })
  }
  return this
},
```

删除列表中指定的回调。

### 删除回调函数

用 `each` 遍历参数列表，在 `each` 遍历里再有一层 `while` 循环，循环的终止条件如下：

```javascript
(index = $.inArray(arg, list, index)) > -1
```

`$.inArray()` 最终返回的是数组项在数组中的索引值，如果不在数组中，则返回 `-1`，所以这个判断是确定回调函数存在于列表中。关于 `$.inArray` 的分析，见《[读zepto源码之工具函数](https://github.com/yeyuqiudeng/reading-zepto/blob/4143f028beff94ce3834e41620fdea48b764301c/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md#inarray)》。

然后调用 `splice` 删除 `list` 中对应索引值的数组项，用 `while` 循环是确保列表中有重复的回调函数都会被删除掉。

### 修正回调任务控制变量

```javascript
if (firing) {
  if (index <= firingLength) --firingLength
  if (index <= firingIndex) --firingIndex
}
```

如果回调任务正在执行中，因为回调列表的长度已经有了变化，需要修正回调任务的控制参数。

如果 `index <= firingLength` ，即回调函数在当前的回调任务中，将回调任务数减少 `1` 。

如果 `index <= firingIndex` ，即在正在执行的回调函数前，将正在执行函数的索引值减少 `1` 。

这样做是防止回调函数执行到最后时，没有找到对应的任务执行。

## .fireWith

```javascript
fireWith: function(context, args) {
  if (list && (!fired || stack)) {
    args = args || []
    args = [context, args.slice ? args.slice() : args]
    if (firing) stack.push(args)
    else fire(args)
      }
  return this
},
```

以指定回调函数的上下文的方式来触发回调函数。

`fireWith` 接收两个参数，第一个参数 `context` 为上下文对象，第二个 `args` 为参数列表。

`fireWith` 后续执行的条件是列表存在并且回调列表没有执行过或者 `stack` 存在（可为空数组），这个要注意，后面讲 `disable` 方法和 `lock` 方法区别的时候，这是一个很重要的判断条件。

```javascript
args = args || []
args = [context, args.slice ? args.slice() : args]
```

先将 `args` 不存在时，初始化为数组。

再重新组合成新的变量 `args` ，这个变量的第一项为上下文对象 `context` ，第二项为参数列表，调用 `args.slice` 是对数组进行拷贝，因为 `memory` 会储存上一次执行的上下文对象及参数，应该是怕外部对引用的更改的影响。

```javascript
if (firing) stack.push(args)
else fire(args)
```

如果回调正处在触发的状态，则将上下文对象和参数先储存在 `stack` 中，从内部函数 `fire` 的分析中可以得知，回调函数执行完毕后，会从 `stack` 中将 `args` 取出，再触发 `fire` 。

否则，触发 `fire`，执行回调函数列表中的回调函数。

`add` 和 `remove` 都要判断 `firing` 的状态，来修正回调任务控制变量，`fire` 方法也要判断 `firing` ，来判断是否需要将 `args` 存入 `stack` 中，但是 `javascript` 是单线程的，照理应该不会出现在触发的同时 `add` 或者 `remove`  或者再调用 `fire` 的情况。

## .fire()

```javascript
fire: function() {
  return Callbacks.fireWith(this, arguments)
},
```

 `fire` 方法，用得最多，但是却非常简单，调用的是 `fireWidth` 方法，上下文对象是 `this` 。

## .has()

```javascript
has: function(fn) {
  return !!(list && (fn ? $.inArray(fn, list) > -1 : list.length))
},
```

`has` 有两个作用，如果有传参时，用来查测所传入的 `fn` 是否存在于回调列表中，如果没有传参时，用来检测回调列表中是否已经有了回调函数。

```javascript
fn ? $.inArray(fn, list) > -1 : list.length
```

这个三元表达式前面的是判断指定的 `fn` 是否存在于回调函数列表中，后面的，如果 `list.length` 大于 `0` ，则回调列表已经存入了回调函数。

## .empty()

```javascript
empty: function() {
  firingLength = list.length = 0
  return this
},
```

`empty` 的作用是清空回调函数列表和正在执行的任务，但是 `list` 还存在，还可以向 `list` 中继续添加回调函数。

## .disable()

```javascript
disable: function() {
  list = stack = memory = undefined
  return this
},
```

`disable` 是禁用回调函数，实质是将回调函数列表置为 `undefined` ，同时也将 `stack` 和 `memory` 置为 `undefined` ，调用 `disable` 后，`add` 、`remove` 、`fire` 、`fireWith` 等方法不再生效，这些方法的首要条件是 `list` 存在。

## .disabled()

```javascript
disabled: function() {
  return !list
},
```

回调是否已经被禁止，其实就是检测 `list` 是否存在。

## .lock()

```javascript
lock: function() {
  stack = undefined
  if (!memory) Callbacks.disable()
  return this
},
```

锁定回调列表，其实是禁止 `fire` 和 `fireWith` 的执行。

其实是将 `stack` 设置为 `undefined` ， `memory` 不存在时，调用的是 `disable` 方法，将整个列表清空。效果等同于禁用回调函数。`fire` 和 `add` 方法都不能再执行。

### .lock() 和 .disable() 的区别

为什么 `memory` 存在时，`stack` 为 `undefined` 就可以将列表的 `fire` 和 `fireWith` 禁用掉呢？在上文的 `fireWith` 中，我特别提到了 `!fired || stack` 这个判断条件。在 `stack` 为 `undefined` 时，`fireWith` 的执行条件看 `fired` 这个条件。如果回调列表已经执行过， `fired` 为 `true` ，`fireWith` 不会再执行。如果回调列表没有执行过，`memory` 为 `undefined` ，会调用 `disable` 方法禁用列表，`fireWith` 也不能执行。

所以，`disable` 和 `lock` 的区别主要是在 `memory` 模式下，回调函数触发过后，`lock` 还可以调用 `add` 方法，向回调列表中添加回调函数，添加完毕后会立刻用 `memory` 的上下文和参数触发回调函数。

## .locked()

```javascript
locked: function() {
  return !stack
},
```

回调列表是否被锁定。

其实就是检测 `stack` 是否存在。

## .fired()

```javascript
fired: function() {
  return !!fired
}
```

回调列表是否已经被触发过。

回调列表触发一次后 `fired` 就会变为 `true`，用 `!!` 的目的是将 `undefined` 转换为 `false` 返回。

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

[署名-非商业性使用-禁止演绎 4.0 国际 (CC BY-NC-ND 4.0)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://user-gold-cdn.xitu.io/2017/5/30/76626b0be42083d36b36f4a117dc1873) 

作者：对角另一面