# 读Zepto源码之集合元素查找

这篇依然是跟 `dom` 相关的方法，侧重点是跟集合元素查找相关的方法。

读Zepto源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## 内部方法

之前有一章《[读Zepto源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md)》是专门解读 `zepto` 中没有提供给外部使用的内部方法的，但是有几个涉及到 `dom` 的方法没有解读，这里先将本章用到的方法解读一下。

### matches

```javascript
zepto.matches = function(element, selector) {
  if (!selector || !element || element.nodeType !== 1) return false
  var matchesSelector = element.matches || element.webkitMatchesSelector ||
      element.mozMatchesSelector || element.oMatchesSelector ||
      element.matchesSelector
  if (matchesSelector) return matchesSelector.call(element, selector)
  // fall back to performing a selector:
  var match, parent = element.parentNode,
      temp = !parent
  if (temp)(parent = tempParent).appendChild(element)
    match = ~zepto.qsa(parent, selector).indexOf(element)
    temp && tempParent.removeChild(element)
    return match
}
```

`matches` 方法用于检测元素( `element` )是否匹配特定的选择器( `selector` )。

浏览器也有原生的 `matches` 方法，但是要到IE9之后才支持。具体见文档：[Element.matches()](https://developer.mozilla.org/en-US/docs/Web/API/Element/matches)

```javascript
if (!selector || !element || element.nodeType !== 1) return false
```

这段是确保 `selector` 和 `element` 两个参数都有传递，并且 `element` 参数的 `nodeType` 为 `ELEMENT_NODE` ，如何条件不符合，返回 `false`

```javascript
 var matchesSelector = element.matches || element.webkitMatchesSelector ||
     element.mozMatchesSelector || element.oMatchesSelector ||
     element.matchesSelector
 if (matchesSelector) return matchesSelector.call(element, selector)
```

这段是检测浏览器是否原生支持 `matches` 方法，或者支持带私有前缀的 `matches` 方法，如果支持，调用原生的 `matches` ，并将结果返回。

```javascript
var match, parent = element.parentNode,
    temp = !parent
if (temp)(parent = tempParent).appendChild(element)
  match = ~zepto.qsa(parent, selector).indexOf(element)
  temp && tempParent.removeChild(element)
  return match
```

如果原生的方法不支持，则回退到用选择器的方法来检测。

这里定义了三个变量，其中 `parent` 用来存放 `element` 的父节点， `temp` 用来判断 `element` 是否有父元素。值为 `temp = !parent` ，如果 `element` 存在父元素，则 `temp` 的值为 `false` 。

首先判断是否存在父元素，如果父元素不存在，则 `parent = tempParent` ，`tempParent` 已经由一个全局变量来定义，为 `tempParent = document.createElement('div')` ，其实就是一个 `div` 空节点。然后将 `element` 插入到空节点中。

然后，查找 `parent` 中所有符合选择器 `selector` 的元素集合，再找出当前元素 `element` 在集合中的索引。

```javascript
zepto.qsa(parent, selector).indexOf(element)
```

再对索引进行取反操作，这样索引值为 `0` 的值就变成了 `-1`，是 `-1` 的返回的是 `0`，这样就确保了跟 `matches` 的表现一致。

其实我有点不太懂的是，为什么不跟原生一样，返回 `boolean` 类型的值呢？明明通过 `zepto.qsa(parent, selector).indexOf(element) > -1` 就可以做到了，接口表现一致不是更好吗？

最后还有一步清理操作：

```javascript
temp && tempParent.removeChild(element)
```

将空接点的子元素清理点，避免污染。

### children

```javascript
function children(element) {
  return 'children' in element ?
    slice.call(element.children) :
  $.map(element.childNodes, function(node) { if (node.nodeType == 1) return node })
}
```

`children` 方法返回的是 `element` 的子元素集合。

浏览器也有原生支持元素 `children` 属性，也要到IE9以上才支持，见文档[ParentNode.children](https://developer.mozilla.org/en-US/docs/Web/API/ParentNode/children)

如果检测到浏览器不支持，则降级用 `$.map` 方法，获取 `element` 的 `childNodes` 中 `nodeType` 为 `ELEMENT_NODE` 的节点。因为 `children` 返回的只是元素节点，但是 `childNodes` 返回的除元素节点外，还包含文本节点、属性等。

这里用到的 `$.map` 跟数组的原生方法 `map` 表现有区别，关于 `$.map` 的具体实现，已经在《[读zepto源码之工具函数](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md#map)》解读过了。

### filtered

```javascript
function filtered(nodes, selector) {
  return selector == null ? $(nodes) : $(nodes).filter(selector)
}
```

将匹配指定选择器的元素从集合中过滤出来。

如果没有指定 `selector` ，则将集合包裹成 `zepto` 对象全部返回，否则调用 `filter` 方法，过滤出符合条件的元素返回。`filter` 方法下面马上讲到。

## 元素方法

这里的方法都是 `$.fn` 中提供的方法。

### .filter()

```javascript
filter: function(selector) {
  if (isFunction(selector)) return this.not(this.not(selector))
  return $(filter.call(this, function(element) {
    return zepto.matches(element, selector)
  }))
}
```

`filter` 是查找符合条件的元素集合。

参数 `selector` 可以为 `Function` 或者选择器，当为 `Function` 时，调用的其实调用了两次 `not` 方法，负负得正。关于 `not` 方法，下面马上会看到。

当为一般的选择器时，调用的是`filter` 方法，`filter` 的回调函数调用了 `matches` ，将符合 `selector` 的元素返回，并包装成 `zepto` 对象返回。

### .not()

```javascript
not: function(selector) {
  var nodes = []
  if (isFunction(selector) && selector.call !== undefined)
    this.each(function(idx) {
      if (!selector.call(this, idx)) nodes.push(this)
        })
    else {
      var excludes = typeof selector == 'string' ? this.filter(selector) :
      (likeArray(selector) && isFunction(selector.item)) ? slice.call(selector) : $(selector)
      this.forEach(function(el) {
        if (excludes.indexOf(el) < 0) nodes.push(el)
          })
    }
  return $(nodes)
}
```

`not` 方法是将集合中不符合条件的元素查找出来。

`not` 方法的方法有三种调用方式：

```javascript
not(selector)  ⇒ collection
not(collection)  ⇒ collection
not(function(index){ ... })  ⇒ collection
```

当 `selector` 为 `Function` ，并且有 `call` 方法时（`isFunction(selector) && selector.call !== undefined`），相关的代码如下：

```javascript
this.each(function(idx) {
  if (!selector.call(this, idx)) nodes.push(this)
    })
```

调用 `each` 方法，并且在 `selector` 函数中，可以访问到当前的元素和元素的索引。如果 `selector` 函数的值取反后为 `true`，则将相应的元素放入 `nodes` 数组中。

当 `selector` 不为 `Function` 时， 定义了一个变量 `excludes` ，这个变量来用接收需要排除的元素集合。接下来又是一串三元表达式（zepto的特色啊）

```javascript
typeof selector == 'string' ? this.filter(selector)
```

当 `selector` 为 `string` 时，调用 `filter` ，找出所有需要排除的元素

```javascript
(likeArray(selector) && isFunction(selector.item)) ? slice.call(selector) : $(selector)
```

这段我刚开始看时，有点困惑，主要是不明白 `isFunction(selector.item)` 这个判断条件，后来查了MDN文档[HTMLCollection.item](https://developer.mozilla.org/en-US/docs/Web/API/HTMLCollection/item)，才明白 `item` 是 `HTMLCollection` 的一个方法，这个三元表达式的意思是，如果是 `HTMLCollection` ，则调用 `slice.call` 得到一个纯数组，否则返回 `zepto` 对象。

```javascript
this.forEach(function(el) {
	if (excludes.indexOf(el) < 0) nodes.push(el)
})
```

遍历集合，如果元素不在需要排除的元素集合中，将该元素 `push` 进 `nodes` 中。

`not` 方法最终返回的也是 `zepto` 对象。

### .is()

```javascript
is: function(selector) {
  return this.length > 0 && zepto.matches(this[0], selector)
}
```

判断集合中的第一个元素是否匹配指定的选择器。

代码也比较简单了，选判断集合不为空，再调用 `matches` 看第一个元素是否匹配。

### .find()

```javascript
find: function(selector) {
  var result, $this = this
  if (!selector) result = $()
  else if (typeof selector == 'object')
    result = $(selector).filter(function() {
      var node = this
      return emptyArray.some.call($this, function(parent) {
        return $.contains(parent, node)
      })
    })
    else if (this.length == 1) result = $(zepto.qsa(this[0], selector))
    else result = this.map(function() { return zepto.qsa(this, selector) })
    return result
}
```

`find` 是查找集合中符合选择器的所有后代元素，如果给定的是 `zepto` 对象或者 `dom` 元素，则只有他们在当前的集合中时，才返回。

`fid` 有三种调用方式，如下：

```javascript
find(selector)   ⇒ collection
find(collection)   ⇒ collection
find(element)   ⇒ collection
```

```javascript
if (!selector) result = $()
```

如果不传参时，返回的是空的 `zepto` 对象。

```javascript
else if (typeof selector == 'object')
  result = $(selector).filter(function() {
    var node = this
    return emptyArray.some.call($this, function(parent) {
      return $.contains(parent, node)
    })
  })
```

如果传参为 `object` 时，也就是 `zepto` 对象`collection` 和`dom` 节点 `element` 时，先将 `selector` 包裹成 `zepto` 对象，然后对这个对象过滤，返回当前集合子节点中所包含的元素（`$.contains(parent, node)`）。

```javascript
else if (this.length == 1) result = $(zepto.qsa(this[0], selector))
```

如果当前的集合只有一个元素时，直接调用 `zepto.qsa` 方法，取出集合的第一个元素 `this[0]` 作为 `qsa` 的第一个参数。关于 `qsa` 方法，已经在《[读Zepto源码之神奇的$](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E7%A5%9E%E5%A5%87%E7%9A%84%24.md#zepto的css选择器-zeptoqsa)》分析过了。其实就是获取第一个元素的所有后代元素。

```javascript
else result = this.map(function() { return zepto.qsa(this, selector) })
```

否则，调用 `map` 方法，对集合中每个元素都调用 `qsa` 方法，获取所有元素的后代元素。这个条件其实可以与上一个条件合并的，分开应该是为了性能的考量。

### .has()

```javascript
has: function(selector) {
  return this.filter(function() {
    return isObject(selector) ?
      $.contains(this, selector) :
    $(this).find(selector).size()
  })
},
```

判断集合中是否有包含指定条件的子元素，将符合条件的元素返回。

有两种调用方式

```javascript
has(selector)   ⇒ collection
has(node)   ⇒ collection
```

参数可以为选择器或者节点。

`has` 其实调用的是 `filter` 方法，这个方法上面已经解读过了。`filter` 的回调函数中根据参数的不同情况，调用了不同的方法。

`isObject(selector)` 用来判断 `selector` 是否为 `node` 节点，如果为 `node` 节点，则调用 `$.contains` 方法，该方法已经在《[读Zepto源码之工具函数](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md#contains)》说过了。

如果为选择器，则调用 `find` 方法，然后再调用 `size` 方法，`size` 方法返回的是集合中元素的个数。这个在《[读Zepto源码之集合操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E6%93%8D%E4%BD%9C.md#size)》有讲过，如果集合个数大于零，则表示满足条件。

### .eq()

```javascript
eq: function(idx) {
  return idx === -1 ? this.slice(idx) : this.slice(idx, +idx + 1)
},
```

获取集合中指定的元素。

这里调用了 `slice` 方法，这个方法在上一篇《[读Zepto源码之集合操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E6%93%8D%E4%BD%9C.md#slice)》已经说过了。如果 `idx` 为 `-1` 时，直接调用 `this.slice(idx)` ，即取出最后一个元素，否则取 `idx` 至 `idx + 1` 之间的元素，也就是每次只取一个元素。`+idx+1` 前面的 `+` 号其实是类型转换，确保 `idx` 在做加法的时候为 `Number` 类型。

### .first()

```javascript
first: function() {
  var el = this[0]
  return el && !isObject(el) ? el : $(el)
},
```

`first` 是取集合中第一个元素，这个方法很简单，用索引 `0` 就可以取出来了，也就是 `this[0]` 。

`el && !isObject(el)` 用来判断是否为 `zepto` 对象，如果不是，用 `$(el)` 包裹，确保返回的是 `zepto` 对象。

### .last()

```javascript
last: function() {
  var el = this[this.length - 1]
  return el && !isObject(el) ? el : $(el)
},
```

`last` 是取集合中最后一个元素，这个的原理跟 `first` 一样，只不过变成了取索引值为 `this.length - 1` ，也就是最后的元素。

### .closest()

```javascript
closest: function(selector, context) {
  var nodes = [],
      collection = typeof selector == 'object' && $(selector)
  this.each(function(_, node) {
    while (node && !(collection ? collection.indexOf(node) >= 0 : zepto.matches(node, selector)))
      node = node !== context && !isDocument(node) && node.parentNode
      if (node && nodes.indexOf(node) < 0) nodes.push(node)
        })
  return $(nodes)
},
```

从元素本身向上查找，返回最先符合条件的元素。

这个方法也有三种调用方式

```javascript
closest(selector, [context])   ⇒ collection
closest(collection)   ⇒ collection 
closest(element)   ⇒ collection 
```

如果指定了 `zepto` 集合或者 `element` ，则只返回匹配给定集合或 `element` 的元素。

```javascript
collection = typeof selector == 'object' && $(selector)
```

这段是判断 `selector` 是否为 `collection` 或 `element` ，如果是，则统一转化为 `zepto` 集合。

然后对集合遍历，在 `each` 遍历里针对集合中每个 `node` 节点，都用 `while` 语句，向上查找符合条件的元素。

```javascript
node && !(collection ? collection.indexOf(node) >= 0 : zepto.matches(node, selector))
```

这段是 `while` 语句的终止条件。 `node` 节点必须存在，如果 `selector` 为 `zepto` 集合或者 `element` ，也即 `collection` 存在， 则要找到存在于 `collection` 中的节点（`collection.indexOf(node) >= 0`）， 否则，节点要匹配指定的选择器（`zepto.matches(node, selector)`）

在 `while` 循环中，是向上逐级查找节点的过程：

```javascript
node = node !== context && !isDocument(node) && node.parentNode
```

当前 `node` 不为指定的上下文 `context` 并且不为 `document` 节点时，向上查找（`node.parentNode`）

```javascript
if (node && nodes.indexOf(node) < 0) nodes.push(node)
```

`while` 循环完毕后，如果 `node` 节点存在，并且 `nodes` 中还不存在 `node` ，则将 `node` push 进 `nodes` 中。

最后返回 `zepto` 集合。

### .pluck()

```javascript
pluck: function(property) {
  return $.map(this, function(el) { return el[property] })
},
```

返回集合中所有元素指定的属性值。

这个方法很简单，就是对当前集合遍历，然后取元素指定的 `property` 值。

### .parents()

```javascript
parents: function(selector) {
  var ancestors = [],
      nodes = this
  while (nodes.length > 0)
    nodes = $.map(nodes, function(node) {
      if ((node = node.parentNode) && !isDocument(node) && ancestors.indexOf(node) < 0) {
        ancestors.push(node)
        return node
      }
    })
    return filtered(ancestors, selector)
},
```

返回集合中所有元素的所有祖先元素。

`nodes` 的初始值为当前集合，`while` 循环的条件为集合不为空。

使用 `map` 遍历 `nodes` ，将 `node` 重新赋值为自身的父级元素，如果父级元素存在，并且不是 `document` 元素，而且还不存在于 `ancestors` 中时，将 `node` 存入保存祖先元素的 `ancestors` 中，并且 `map` 回调的返回值是 `node` ，组成新的集合赋值给 `nodes` ，直到所有的祖先元素遍历完毕，就可以退出 `while` 循环。

最后，调用上面说到的 `filtered` 方法，找到符合 `selector` 的祖先元素。

### .parent()

```javascript
parent: function(selector) {
  return filtered(uniq(this.pluck('parentNode')), selector)
},
```

返回集合中所有元素的父级元素。

`parents` 返回的是所有祖先元素，而 `parent` 返回只是父级元素。

首先调用的是 `this.pluck('parentNode')` ，获取所有元素的祖先元素，然后调用 `uniq` 对集合去重，最后调用 `filtered` ，返回匹配 `selector` 的元素集合。

### .children()

```javascript
children: function(selector) {
  return filtered(this.map(function() { return children(this) }), selector)
},
```

返回集合中所有元素的子元素。

首先对当前集合遍历，调用内部方法 `children` 获取当前元素的子元素组成新的数组，再调用 `filtered` 方法返回匹配 `selector` 的元素集合。

### .contents()

```javascript
contents: function() {
  return this.map(function() { return this.contentDocument || slice.call(this.childNodes) })
},
```

这个方法类似于 `children` ，不过 `children` 对 `childNodes` 进行了过滤，只返回元素节点。`contents` 还返回文本节点和注释节点。也返回 `iframe` 的 `contentDocument`

### .siblings()

```javascript
siblings: function(selector) {
  return filtered(this.map(function(i, el) {
    return filter.call(children(el.parentNode), function(child) { return child !== el })
  }), selector)
},
```

获取所有集合中所有元素的兄弟节点。

获取兄弟节点的思路也很简单，对当前集合遍历，找到当前元素的父元素`el.parentNode`，调用 `children` 方法，找出父元素的子元素，将子元素中与当前元素不相等的元素过滤出来即是其兄弟元素了。

最后调用 `filtered` 来过滤出匹配 `selector` 的兄弟元素。

### .prev()

```javascript
prev: function(selector) { return $(this.pluck('previousElementSibling')).filter(selector || '*') },
```

获取集合中每个元素的前一个兄弟节点。

这个方法也很简单，调用 `pluck` 方法，获取元素的 `previousElementSibling` 属性，即为元素的前一个兄弟节点。再调用 `filter` 返回匹配 `selector` 的元素，最后包裹成 `zepto` 对象返回

### .next()

```javascript
next: function(selector) { return $(this.pluck('nextElementSibling')).filter(selector || '*') },
```

`next` 方法跟 `prev` 方法类似，只不过取的是 `nextElementSibling` 属性，获取的是每个元素的下一个兄弟节点。

### .index()

```javascript
index: function(element) {
  return element ? this.indexOf($(element)[0]) : this.parent().children().indexOf(this[0])
},
```

返回指定元素在当前集合中的位置（`this.indexOf($(element)[0])`），如果没有给出 `element` ，则返回当前鲜红在兄弟元素中的位置。`this.parent().children()` 查找的是兄弟元素。

## 系列文章

1. [读Zepto源码之代码结构](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E4%BB%A3%E7%A0%81%E7%BB%93%E6%9E%84.md)
2. [读 Zepto 源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md)
3. [读Zepto源码之工具函数](https://github.com/yeyuqiudeng/reading-zepto/blob/a4d6ad99c57047beae2b652b4d2cbb380599a524/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md)
4. [读Zepto源码之神奇的$](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E7%A5%9E%E5%A5%87%E7%9A%84%24.md)
5. [读Zepto源码之集合操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E6%93%8D%E4%BD%9C.md)

## 参考

* [Element.matches()](https://developer.mozilla.org/en-US/docs/Web/API/Element/matches)
* [ParentNode.children](https://developer.mozilla.org/en-US/docs/Web/API/ParentNode/children)
* [Node.childNodes](https://developer.mozilla.org/en-US/docs/Web/API/Node/childNodes)
* [HTMLCollection.item](https://developer.mozilla.org/en-US/docs/Web/API/HTMLCollection/item)

## License

[![License: CC BY-NC-ND 4.0](https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://segmentfault.com/img/bVCJ55?w=430&h=430) 

作者：对角另一面