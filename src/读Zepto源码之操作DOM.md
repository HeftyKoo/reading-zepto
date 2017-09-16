# 读Zepto源码之操作DOM

这篇依然是跟 `dom` 相关的方法，侧重点是操作 `dom` 的方法。

读Zepto源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## .remove()

```javascript
remove: function() {
  return this.each(function() {
    if (this.parentNode != null)
      this.parentNode.removeChild(this)
    })
},
```

删除当前集合中的元素。

如果父节点存在时，则用父节点的 `removeChild` 方法来删掉当前的元素。

## 相似方法生成器

`zepto` 中 `after`、 `prepend`、 `before`、 `append`、`insertAfter`、 `insertBefore`、 `appendTo` 和 `prependTo` 都是通过这个相似方法生成器生成的。

### 定义容器

```javascript
adjacencyOperators = ['after', 'prepend', 'before', 'append']
```

首先，定义了一个相似操作的数组，注意数组里面只有 `after`、 `prepend`、 `before`、 `append` 这几个方法名，后面会看到，在生成这几个方法后，`insertAfter`、 `insertBefore`、 `appendTo` 和 `prependTo` 会分别调用前面生成的几个方法。

### 辅助方法traverseNode

```javascript
function traverseNode(node, fun) {
  fun(node)
  for (var i = 0, len = node.childNodes.length; i < len; i++)
    traverseNode(node.childNodes[i], fun)
}
```

这个方法递归遍历 `node` 的子节点，将节点交由回调函数 `fun` 处理。这个辅助方法在后面会用到。

### 核心源码

```javascript
adjacencyOperators.forEach(function(operator, operatorIndex) {
  var inside = operatorIndex % 2 //=> prepend, append

  $.fn[operator] = function() {
    // arguments can be nodes, arrays of nodes, Zepto objects and HTML strings
    var argType, nodes = $.map(arguments, function(arg) {
      var arr = []
      argType = type(arg)
      if (argType == "array") {
        arg.forEach(function(el) {
          if (el.nodeType !== undefined) return arr.push(el)
          else if ($.zepto.isZ(el)) return arr = arr.concat(el.get())
          arr = arr.concat(zepto.fragment(el))
        })
        return arr
      }
      return argType == "object" || arg == null ?
        arg : zepto.fragment(arg)
    }),
        parent, copyByClone = this.length > 1
    if (nodes.length < 1) return this

    return this.each(function(_, target) {
      parent = inside ? target : target.parentNode

      // convert all methods to a "before" operation
      target = operatorIndex == 0 ? target.nextSibling :
      operatorIndex == 1 ? target.firstChild :
      operatorIndex == 2 ? target :
      null

      var parentInDocument = $.contains(document.documentElement, parent)

      nodes.forEach(function(node) {
        if (copyByClone) node = node.cloneNode(true)
        else if (!parent) return $(node).remove()

        parent.insertBefore(node, target)
        if (parentInDocument) traverseNode(node, function(el) {
          if (el.nodeName != null && el.nodeName.toUpperCase() === 'SCRIPT' &&
              (!el.type || el.type === 'text/javascript') && !el.src) {
            var target = el.ownerDocument ? el.ownerDocument.defaultView : window
            target['eval'].call(target, el.innerHTML)
          }
        })
          })
    })
  }
```

### 调用方式

在分析之前，先看看这几个方法的用法：

```javascript
after(content)
prepend(content)
before(content)
append(content)
```

参数 `content` 可以为 `html` 字符串，`dom` 节点，或者节点组成的数组。`after` 是在每个集合元素后插入 `content` ， `before` 正好相反，在每个集合元素前插入 `content`，`prepend` 是在每个集合元素的初始位置插入 `content`， `append` 是在每个集合元素的末尾插入 `content`。`before` 和 `after` 插入的 `content` 在元素的外部，而 `prepend` 和 `append` 插入的 `content` 在元素的内部，这是需要注意的。

### 将参数 `content` 转换成 `node` 节点数组

```javascript
var inside = operatorIndex % 2 //=> prepend, append
```

遍历 `adjacencyOperators`，得到对应的方法名 `operator` 和方法名在数组中的索引 `operatorIndex`。

定义了一个 `inside` 变量，当 `operatorIndex` 为奇数时，`inside` 的值为 `true`，也就是 `operator` 的值为 `prepend` 或 `append` 时，`inside` 的值为 `true` 。这个可以用来区分 `content` 是插入到元素内部还是外部的方法。

`$.fn[operator]` 即为 `$.fn` 对象设置对应的属性值（方法名）。

```javascript
var argType, nodes = $.map(arguments, function(arg) {
  var arr = []
  argType = type(arg)
  if (argType == "array") {
    arg.forEach(function(el) {
      if (el.nodeType !== undefined) return arr.push(el)
      else if ($.zepto.isZ(el)) return arr = arr.concat(el.get())
      arr = arr.concat(zepto.fragment(el))
    })
    return arr
  }
  return argType == "object" || arg == null ?
    arg : zepto.fragment(arg)
}),
```

变量 `argType` 用来保存变量变量的类型，也即 `content` 的类型。`nodes` 是根据 `content` 转换后的 `node` 节点数组。

这里用了 `$.map` `arguments` 的方式来获取参数 `content` ，这里只有一个参数，这什么不用 `arguments[0]` 来获取呢？这是因为 `$.map` 可以将数组进行展平，具体的实现看这里《[读zepto源码之工具函数](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md#map)》。

首先用内部函数 `type` 来获取参数的类型，关于 `type` 的实现，在《[读Zepto源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md#type)》 已经作过分析。

如果参数 `content` ，也即 `arg` 的类型为数组时，遍历 `arg` ，如果数组中的元素存在 `nodeType` 属性，则断定为 `node` 节点，就将其 `push` 进容器 `arr` 中；如果数组中的元素为 `zepto` 对象（用 `$.zepto.isZ` 判断，该方法已经在《[读Zepto源码之神奇的$](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E7%A5%9E%E5%A5%87%E7%9A%84%24.md#zeptoisz)》有过分析），不传参调用 `get` 方法，返回的是一个数组，然后调用数组的 `concat` 方法合并数组，`get` 方法在《[读Zepto源码之集合操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E6%93%8D%E4%BD%9C.md#get)》有过分析；否则，为 `html` 字符串，调用 `zepto.fragment` 处理，并将返回的数组合并，``zepto.fragment` 在《[读Zepto源码之神奇的$](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E7%A5%9E%E5%A5%87%E7%9A%84%24.md#zeptofragment)》中有过分析。

如果参数类型为 `object` （即为 `zepto` 对象）或者 `null` ，则直接返回。

否则为 `html` 字符串，调用 `zepto.fragment` 处理。

```javascript
parent, copyByClone = this.length > 1
if (nodes.length < 1) return this
```

这里还定义了 `parent` 变量，用来保存 `content` 插入的父节点；当集合中元素的数量大于 `1` 时，变量 `copyByClone` 的值为 `true` ，这个变量的作用后面再说。

如果 `nodes` 的数量比 `1` 小，也即需要插入的节点为空时，不再作后续的处理，返回 `this` ，以便可以进行链式操作。

### 用 `insertBefore` 来模拟所有操作

```javascript
return this.each(function(_, target) {
  parent = inside ? target : target.parentNode

  // convert all methods to a "before" operation
  target = operatorIndex == 0 ? target.nextSibling :
  operatorIndex == 1 ? target.firstChild :
  operatorIndex == 2 ? target :
  null

  var parentInDocument = $.contains(document.documentElement, parent)
  ...
})
```

对集合进行 `each` 遍历

```javascript
parent = inside ? target : target.parentNode
```

如果 `node` 节点需要插入目标元素 `target` 的内部，则 `parent` 设置为目标元素 `target`，否则设置为当前元素的父元素。

```javascript
target = operatorIndex == 0 ? target.nextSibling :
  operatorIndex == 1 ? target.firstChild :
  operatorIndex == 2 ? target :
  null
```

这段是将所有的操作都用 `dom` 原生方法 `insertBefore` 来模拟。 如果 `operatorIndex == 0` 即为 `after` 时，`node` 节点应该插入到目标元素 `target` 的后面，即 `target` 的下一个兄弟元素的前面；当 `operatorIndex == 1` 即为 `prepend` 时，`node` 节点应该插入到目标元素的开头，即 `target` 的第一个子元素的前面；当 `operatorIndex == 2` 即为 `before` 时，`insertBefore` 刚好与之对应，即为元素本身。当 `insertBefore` 的第二个参数为 `null` 时，`insertBefore` 会将 `node` 插入到子节点的末尾，刚好与 `append` 对应。具体见文档：[Node.insertBefore()](https://developer.mozilla.org/en-US/docs/Web/API/Node/insertBefore)

```javascript
var parentInDocument = $.contains(document.documentElement, parent)
```

调用 `$.contains` 方法，检测父节点 `parent` 是否在 `document` 中。`$.contains` 方法在《[读zepto源码之工具函数](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md#contains)》中已有过分析。

### 将 `node` 节点数组插入到元素中

```javascript
nodes.forEach(function(node) {
  if (copyByClone) node = node.cloneNode(true)
  else if (!parent) return $(node).remove()

  parent.insertBefore(node, target)
  ...
})
```

如果需要复制节点时（即集合元素的数量大于 `1` 时），用 `node` 节点方法 `cloneNode` 来复制节点，参数 `true` 表示要将节点的子节点和属性等信息也一起复制。为什么集合元素大于 `1` 时需要复制节点呢？因为 `insertBefore` 插入的是节点的引用，对集合中所有元素的遍历操作，如果不克隆节点，每个元素所插入的引用都是一样的，最后只会将节点插入到最后一个元素中。

如果父节点不存在，则将 `node` 删除，不再进行后续操作。

将节点用 `insertBefore` 方法插入到元素中。

### 处理 `script` 标签内的脚本 

```javascript
if (parentInDocument) traverseNode(node, function(el) {
  if (el.nodeName != null && el.nodeName.toUpperCase() === 'SCRIPT' &&
      (!el.type || el.type === 'text/javascript') && !el.src) {
    var target = el.ownerDocument ? el.ownerDocument.defaultView : window
    target['eval'].call(target, el.innerHTML)
  }
})
```

如果父元素在 `document` 内，则调用 `traverseNode` 来处理 `node` 节点及 `node` 节点的所有子节点。主要是检测 `node` 节点或其子节点是否为不指向外部脚本的 `script` 标签。

```javascript
el.nodeName != null && el.nodeName.toUpperCase() === 'SCRIPT'
```

这段用来判断是否为 `script` 标签，通过 `node` 的 `nodeName` 属性是否为 `script` 来判断。

```javascript
!el.type || el.type === 'text/javascript'
```

不存在 `type` 属性，或者 `type` 属性为 `'text/javascript'`。这里表示只处理 `javascript`，因为 `type` 属性不一定指定为 `text/javascript` ，只有指定为 `test/javascript` 或者为空时，才会按照 `javascript` 来处理。见[MDN文档script](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script)

```javascript
!el.src
```

并且不存在外部脚本。

```javascript
var target = el.ownerDocument ? el.ownerDocument.defaultView : window
```

是否存在 `ownerDocument` 属性，`ownerDocument` 返回的是元素的根节点，也即 `document` 对象，`document` 对象的 `defaultView` 属性返回的是 `document` 对象所关联的 `window` 对象，这里主要是处理 `iframe` 里的 `script`，因为在 `iframe` 中有独立的 `window` 对象。如果不存在该属性，则默认使用当前的 `window` 对象。

```javascript
target['eval'].call(target, el.innerHTML)
```

最后调用 `window` 的 `eval` 方法，执行 `script` 中的脚本，脚本用 `el.innerHTML` 取得。

为什么要对 `script` 元素单独进行这样的处理呢？因为出于安全的考虑，脚本通过 `insertBefore` 的方法插入到 `dom` 中时，是不会执行脚本的，所以需要使用 `eval` 来进行处理。

### 生成 `insertAfter`、`prependTo`、`insertBefore` 和 `appendTo` 方法

先来看看这几个方法的调用方式

```javascript
insertAfter(target)
insertBefore(target)
appendTo(target)
prependTo(target)
```

这几个方法都是将集合中的元素插入到目标元素 `target` 中，跟 `after`、`before`、`append` 和 `prepend` 刚好是相反的操作。

他们的对应关系如下：

```javascript
after    => insertAfter
prepend  => prependTo
before   => insertBefore
append   => appendTo
```

因此可以调用相应的方法来生成这些方法。

```javascript
$.fn[inside ? operator + 'To' : 'insert' + (operatorIndex ? 'Before' : 'After')] = function(html) {
  $(html)[operator](this)
  return this
}
```

```javascript
inside ? operator + 'To' : 'insert' + (operatorIndex ? 'Before' : 'After')
```

这段其实是生成方法名，如果是 `prepend` 或 `append` ，则在后面拼接 `To` ，如果是 `Before` 或 `After`，则在前面拼接 `insert`。

```javascript
$(html)[operator](this)
```

简单地反向调用对应的方法，就可以了。

到此，这个相似方法生成器生成了`after`、 `prepend`、 `before`、 `append`、`insertAfter`、 `insertBefore`、 `appendTo` 和 `prependTo` 等八个方法，相当高效。

## .empty()

```javascript
empty: function() {
  return this.each(function() { this.innerHTML = '' })
},
```

`empty` 的作用是将所有集合元素的内容清空，调用的是 `node` 的 `innerHTML` 属性设置为空。

## .replaceWith()

```javascript
replaceWith: function(newContent) {
  return this.before(newContent).remove()
},
```

将所有集合元素替换为指定的内容 `newContent` ， `newContent` 的类型跟 `before` 的参数类型一样。

`replaceWidth` 首先调用 `before` 将 `newContent` 插入到对应元素的前面，再将元素删除，这样就达到了替换的上的。

## .wrapAll()

```javascript
wrapAll: function(structure) {
  if (this[0]) {
    $(this[0]).before(structure = $(structure))
    var children
    // drill down to the inmost element
    while ((children = structure.children()).length) structure = children.first()
    $(structure).append(this)
  }
  return this
},
```

将集合中所有的元素都包裹进指定的结构 `structure` 中。

如果集合元素存在，即 `this[0]` 存在，则进行后续操作，否则返回 `this` ，以进行链式操作。

调用 `before` 方法，将指定结构插入到第一个集合元素的前面，也即所有集合元素的前面

```javascript
while ((children = structure.children()).length) structure = children.first()
```

查找 `structure` 的子元素，如果子元素存在，则将 `structure` 赋值为 `structure` 的第一个子元素，直找到 `structrue` 最深层的第一个子元素为止。

将集合中所有的元素都插入到 `structure` 的末尾，如果 `structure` 存在子元素，则插入到最深层的第一个子元素的末尾。这样就将集合中的所有元素都包裹到 `structure` 内了。

## .wrap()

```javascript
wrap: function(structure) {
  var func = isFunction(structure)
  if (this[0] && !func)
    var dom = $(structure).get(0),
        clone = dom.parentNode || this.length > 1

    return this.each(function(index) {
      $(this).wrapAll(
        func ? structure.call(this, index) :
        clone ? dom.cloneNode(true) : dom
      )
    })
},
```

为集合中每个元素都包裹上指定的结构 `structure`，`structure` 可以为单独元素或者嵌套元素，也可以为 `html` 元素或者 `dom` 节点，还可以为回调函数，回调函数接收当前元素和当前元素在集合中的索引两个参数，返回符合条件的包裹结构。

```javascript
var func = isFunction(structure)
```

 判断 `structure` 是否为函数

```javascript
if (this[0] && !func)
  var dom = $(structure).get(0),
      clone = dom.parentNode || this.length > 1
```

如果集合不为空，并且 `structure` 不为函数，则将 `structure` 转换为 `node` 节点，通过 `$(structure).get(0)` 来转换，并赋给变量 `dom`。如果 `dom` 的 `parentNode` 存在或者集合的数量大于 `1` ，则 `clone` 的值为 `true`。

```javascript
return this.each(function(index) {
  $(this).wrapAll(
  func ? structure.call(this, index) :
  clone ? dom.cloneNode(true) : dom
  )
})
```

对集合进行遍历，调用 `wrapAll` 方法，如果 `structure` 为函数，则将回调函数返回的结果作为参数传给 `wrapAll` ；

否则，如果 `clone` 为 `true` ，则将 `dom` 也即包裹元素的副本传给 `wrapAll` ，否则直接将 `dom` 传给 `wrapAll`。这里传递副本的的原因跟生成器中的一样，也是避免对 `dom` 节点的引用。如果 `dom` 的 `parentNode` 存在时，表明 `dom` 本来就从属于某个节点，如果直接使用 `dom` ，会破坏原来的结构。

## .wrapInner()

```javascript
wrapInner: function(structure) {
  var func = isFunction(structure)
  return this.each(function(index) {
    var self = $(this),
        contents = self.contents(),
        dom = func ? structure.call(this, index) : structure
    contents.length ? contents.wrapAll(dom) : self.append(dom)
  })
},
```

将集合中每个元素的内容都用指定的结构 `structure` 包裹。 `structure` 的参数类型跟 `wrap` 一样。

对集合进行遍历，调用 `contents` 方法，获取元素的内容，`contents` 方法在《[读Zepto源码之集合元素查找](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E5%85%83%E7%B4%A0%E6%9F%A5%E6%89%BE.md#contents)》有过分析。

如果 `structure` 为函数，则将函数返回的结果赋值给 `dom` ，否则将直接将 `structure` 赋值给 `dom`。

如果 `contents.length` 存在，即元素不为空元素，调用 `wrapAll` 方法，将元素的内容包裹在 `dom` 中；如果为空元素，则直接将 `dom` 插入到元素的末尾，也实现了将 `dom` 包裹在元素的内部了。

## .unwrap()

```javascript
unwrap: function() {
  this.parent().each(function() {
    $(this).replaceWith($(this).children())
  })
  return this
},
```

当集合中的所有元素的包裹层去掉，也即将父元素去掉，但是保留父元素的子元素。

实现的方法也很简单，就是遍历当前元素的父元素，将父元素替换为父元素的子元素。

## .clone()

```javascript
clone: function() {
  return this.map(function() { return this.cloneNode(true) })
},
```

每集合中每个元素都创建一个副本，并将副本集合返回。

遍历元素集合，调用 `node` 的原生方法 `cloneNode` 创建副本。要注意，`cloneNode` 不会将元素原来的数据和事件处理程序复制到副本中。

## 系列文章

1. [读Zepto源码之代码结构](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E4%BB%A3%E7%A0%81%E7%BB%93%E6%9E%84.md)
2. [读 Zepto 源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md)
3. [读Zepto源码之工具函数](https://github.com/yeyuqiudeng/reading-zepto/blob/a4d6ad99c57047beae2b652b4d2cbb380599a524/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md)
4. [读Zepto源码之神奇的$](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E7%A5%9E%E5%A5%87%E7%9A%84%24.md)
5. [读Zepto源码之集合操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E6%93%8D%E4%BD%9C.md)
6. [读Zepto源码之集合元素查找](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E5%85%83%E7%B4%A0%E6%9F%A5%E6%89%BE.md)

## 参考

* [Node.insertBefore()](https://developer.mozilla.org/en-US/docs/Web/API/Node/insertBefore)
* [Node.cloneNode()](https://developer.mozilla.org/en-US/docs/Web/API/Node/cloneNode)
* [Zepto源码分析-zepto模块](http://www.cnblogs.com/mominger/p/4369206.html)
* [MDN文档script](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script)
* [Node.ownerDocument](https://developer.mozilla.org/en-US/docs/Web/API/Node/ownerDocument)
* [Document.defaultView](https://developer.mozilla.org/en-US/docs/Web/API/Document/defaultView)

## License

[署名-非商业性使用-禁止演绎 4.0 国际 (CC BY-NC-ND 4.0)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://user-gold-cdn.xitu.io/2017/5/30/76626b0be42083d36b36f4a117dc1873) 

作者：对角另一面