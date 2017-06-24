# 读Zepto源码之属性操作

这篇依然是跟 `dom` 相关的方法，侧重点是操作属性的方法。

读Zepto源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## 内部方法

### setAttribute

```javascript
function setAttribute(node, name, value) {
  value == null ? node.removeAttribute(name) : node.setAttribute(name, value)
}
```

如果属性值 `value` 存在，则调用元素的原生方法 `setAttribute` 设置对应元素的指定属性值，否则调用 `removeAttribute` 删除指定的属性。

### deserializeValue

```javascript
// "true"  => true
// "false" => false
// "null"  => null
// "42"    => 42
// "42.5"  => 42.5
// "08"    => "08"
// JSON    => parse if valid
// String  => self
function deserializeValue(value) {
  try {
    return value ?
      value == "true" ||
      (value == "false" ? false :
       value == "null" ? null :
       +value + "" == value ? +value :
       /^[\[\{]/.test(value) ? $.parseJSON(value) :
       value) :
    value
  } catch (e) {
    return value
  }
}
```

函数的主体又是个很复杂的三元表达式，但是函数要做什么事情，注释已经写得很明白了。

`try catch` 保证出错的情况下依然可以将原值返回。

先将这个复杂的三元表达式拆解下：

```javascript
value ? 相当复杂的表达式返回的值 : value
```

值存在时，就进行相当复杂的三元表达式运算，否则返回原值。

再来看看 `value === "true"` 时的运算

```javascript
value == "true" || (复杂表达式求出的值) 
```

这其实是一个或操作，当 `value === "true"` 时就不执行后面的表达式，直接将 `value === "true"`  的值返回，也就是返回 `true`

再来看 `value === false` 时的求值

```javascript
value == "false" ? false : (其他表达式求出来的值)
```

很明显，`value === "false"` 时，返回的值为 `false`

```javascript
value == "null" ? null : (其他表达式求出来的值)
```

为 `value == "null"` 时， 返回值为 `null`

再来看看数字字符串的判断：

```javascript
 +value + "" == value ? +value : (其他表达式求出来的值)
```

这个判断相当有意思。

`+value` 将 `value` 隐式转换成数字类型，`"42"` 转换成 `42` ，`"08"` 转换成 `8` ，`abc` 会转换成 `NaN` 。`+ ""` 是将转换成数字后的值再转换成字符串。然后再用 `==` 和原值比较。这里要注意，用的是 `==` ，不是 `===` 。左边表达式不用说，肯定是字符串类型，右边的如果为字符串类型，并且和左边的值相等，那表示 `value` 为数字字符串，可以用 `+value` 直接转换成数字。 但是以 `0` 开头的数字字符串如 `"08"` ，经过左边的转换后变成 `"8"`，两个字符串不相等，继续执行后面的逻辑。

如果 `value` 为数字，则左边的字符串会再次转换成数字后再和 `value` 进行比较，左边转换成数字后肯定为 `value` 本身，因此表达式成立，返回一样的数字。

```javascript
/^[\[\{]/.test(value) ? $.parseJSON(value) : value
```

这长长的三元表达式终于被剥得只剩下内衣了。

`/^[\[\{]/` 这个正则是检测 `value` 是否以 `[` 或者 `{` 开头，如果是，则将其作为对象或者数组，执行 `$.parseJSON` 方法反序列化，否则按原值返回。

其实，这个正则不太严谨的，以这两个符号开头的字符串，可能根本不是对象或者数组格式的，序列化可能会出错，这就是一开始提到的 `try catch` 所负责的事了。

## .html()

```javascript
html: function(html) {
  return 0 in arguments ?
    this.each(function(idx) {
    var originHtml = this.innerHTML
    $(this).empty().append(funcArg(this, html, idx, originHtml))
  }) :
  (0 in this ? this[0].innerHTML : null)
},
```

`html` 方法既可以设置值，也可以获取值，参数 `html` 既可以是固定值，也可以是函数。

`html` 方法的主体是一个三元表达式， `0 in arguments` 用来判断方法是否带参数，如果不带参数，则获取值，否则，设置值。

```javascript
(0 in this ? this[0].innerHTML : null)
```

先来看看获取值，`0 in this` 是判断集合是否为空，如果为空，则返回 `null` ，否则，返回的是集合第一个元素的 `innerHTML` 属性值。

```javascript
this.each(function(idx) {
  var originHtml = this.innerHTML
  $(this).empty().append(funcArg(this, html, idx, originHtml))
})
```

知道值怎样获取后，设置也就简单了，要注意一点的是，设置值的时候，集合中每个元素的 `innerHTML` 值都被设置为给定的值。

由于参数 `html` 可以是固定值或者函数，所以先调用内部函数 `funcArg` 来对参数进行处理，`funcArg` 的分析请看 《[读Zepto源码之样式操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E6%A0%B7%E5%BC%8F%E6%93%8D%E4%BD%9C.md#funcarg)》 。

设置的逻辑也很简单，先将当前元素的内容清空，调用的是  `empty` 方法，然后再调用 `append` 方法，插入给定的值到当前元素中。`append` 方法的分析请看《[读Zepto源码之操作DOM](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E6%93%8D%E4%BD%9CDOM.md#相似方法生成器)》

## .text()

```javascript
text: function(text) {
  return 0 in arguments ?
    this.each(function(idx) {
    var newText = funcArg(this, text, idx, this.textContent)
    this.textContent = newText == null ? '' : '' + newText
  }) :
  (0 in this ? this.pluck('textContent').join("") : null)
},
```

`text` 方法用于获取或设置元素的 `textContent` 属性。

先看不传参的情况：

```javascript
(0 in this ? this.pluck('textContent').join("") : null)
```

调用 `pluck` 方法获取每个元素的 `textContent` 属性，并且将结果集合并成字符串。关于 `textContent` 和 `innerText` 的区别，MDN上说得很清楚：

* `textContent` 会获取所有元素的文本，包括 `script` 和 `style` 的元素
* `innerText` 不会将隐藏元素的文本返回
* `innerText` 元素遇到 `style` 时，会重绘

具体参考 [MDN:Node.textContent](https://developer.mozilla.org/en-US/docs/Web/API/Node/textContent)

设置值的逻辑中 `html` 方法差不多，但是在 `newText == null` 时，赋值为 `''` ，否则，转换成字符串。这个转换我有点不太明白， 赋值给 `textContent` 时，会自动转换成字符串，为什么要自己转换一次呢？还有，`textContent` 直接赋值为 `null` 或者 `undefined` ，也会自动转换为 `''` ，为什么还要自己转换一次呢？

## .attr()

```javascript
attr: function(name, value) {
  var result
  return (typeof name == 'string' && !(1 in arguments)) ?
    (0 in this && this[0].nodeType == 1 && (result = this[0].getAttribute(name)) != null ? result : undefined) :
  this.each(function(idx) {
    if (this.nodeType !== 1) return
    if (isObject(name))
    for (key in name) setAttribute(this, key, name[key])
    else setAttribute(this, name, funcArg(this, value, idx, this.getAttribute(name)))
      })
},
```

 `attr` 用于获取或设置元素的属性值。`name` 参数可以为 `object` ，用于设置多组属性值。

判断条件：

```javascript
typeof name == 'string' && !(1 in arguments)
```

参数 `name` 为字符串，排除掉 `name` 为 `object` 的情况，并且第二个参数不存在，在这种情况下，为获取值。

```javascript
(0 in this && this[0].nodeType == 1 && (result = this[0].getAttribute(name)) != null ? result : undefined)
```

获取属性时，要满足几个条件：

1. 集合不为空
2. 集合的第一个元素的 `nodeType` 为 `ELEMENT_NODE`

然后调用元素的原生方法 `getAttribute` 方法来获取第一个元素对应的属性值，如果属性值 `!=null` ，则返回获取到的属性值，否则返回 `undefined` 。

再来看设置值的情况：

```javascript
this.each(function(idx) {
  if (this.nodeType !== 1) return
  if (isObject(name))
  	for (key in name) setAttribute(this, key, name[key])
  else setAttribute(this, name, funcArg(this, value, idx, this.getAttribute(name)))
    })
```

如果元素的 `nodeType` 不为 `ELEMENT_NODE` 时，直接 `return`

当 `name` 为 `object` 时，遍历对象，设置对应的属性

否则，设置给定属性的值。

## .removeAttr()

```javascript
removeAttr: function(name) {
  return this.each(function() {
    this.nodeType === 1 && name.split(' ').forEach(function(attribute) {
      setAttribute(this, attribute)
    }, this)
  })
},
```

删除给定的属性。可以用空格分隔多个属性。

调用的其实是 `setAttribute` 方法，只将元素和需要删除的属性传递进去， `setAttribute` 就会将对应的元素属性删除。

## .prop()

```javascript
propMap = {
  'tabindex': 'tabIndex',
  'readonly': 'readOnly',
  'for': 'htmlFor',
  'class': 'className',
  'maxlength': 'maxLength',
  'cellspacing': 'cellSpacing',
  'cellpadding': 'cellPadding',
  'rowspan': 'rowSpan',
  'colspan': 'colSpan',
  'usemap': 'useMap',
  'frameborder': 'frameBorder',
  'contenteditable': 'contentEditable'
}
prop: function(name, value) {
  name = propMap[name] || name
  return (1 in arguments) ?
    this.each(function(idx) {
    this[name] = funcArg(this, value, idx, this[name])
  }) :
  (this[0] && this[0][name])
},
```

`prop` 也是给元素设置或获取属性，但是跟 `attr` 不同的是， `prop` 设置的是元素本身固有的属性，`attr` 用来设置自定义的属性（也可以设置固有的属性）。

`propMap` 是将一些特殊的属性做一次映射。

`prop` 取值和设置值的时候，都是直接操作元素对象上的属性，不需要调用如 `setAttribute` 的方法。

## .removeProp()

```javascript
removeProp: function(name) {
  name = propMap[name] || name
  return this.each(function() { delete this[name] })
},
```

删除元素固定属性，调用对象的 `delete` 方法就可以了。

## .data()

```javascript
capitalRE = /([A-Z])/g
data: function(name, value) {
  var attrName = 'data-' + name.replace(capitalRE, '-$1').toLowerCase()

  var data = (1 in arguments) ?
      this.attr(attrName, value) :
  this.attr(attrName)

  return data !== null ? deserializeValue(data) : undefined
},
```

`data` 内部调用的是 `attr` 方法，但是给属性名加上了 `data-` 前缀，这也是向规范靠拢。

```javascript
name.replace(capitalRE, '-$1').toLowerCase()
```

稍微解释下这个正则，`capitalRE` 匹配的是大写字母，`replace(capitalRE, '-$1')` 是在大写字母前面加上 `-` 连字符。这整个表达式其实就是将 `name` 转换成 `data-camel-case` 的形式。

```javascript
return data !== null ? deserializeValue(data) : undefined
```

如果 `data` 不严格为 `null` 时，调用 `deserializeValue` 序列化后返回，否则返回 `undefined` 。为什么要用严格等 `null` 来作为判断呢？这个我也不太明白，因为在获取值时，`attr` 方法对不存在的属性返回值为 `undefined` ，用 `!== undefined` 判断会不会更好点呢？这样 `undefined` 根本不需要再走 `deserializeValue` 方法。

## .val()

```javascript
val: function(value) {
  if (0 in arguments) {
    if (value == null) value = ""
    return this.each(function(idx) {
      this.value = funcArg(this, value, idx, this.value)
    })
  } else {
    return this[0] && (this[0].multiple ?
                       $(this[0]).find('option').filter(function() { return this.selected }).pluck('value') :
                       this[0].value)
  }
},
```

获取或设置表单元素的 `value` 值。

如果传参，还是惯常的套路，设置的是元素的 `value` 属性。

否则，获取值，看看获取值的逻辑：

```javascript
return this[0] && (this[0].multiple ? 
                   $(this[0]).find('option').filter(function() { return this.selected }).pluck('value') : 
                   this[0].value)
```

`this[0].multiple ` 判断是否为下拉列表多选，如果是，则找出所有选中的 `option` ，获取选中的 `option` 的 `value` 值返回。这里用到 `pluck` 方法来获取属性，具体的分析见：《[读Zepto源码之集合元素查找](https://github.com/yeyuqiudeng/reading-zepto/blob/6233a527bf8138e6e93359d67a3311b0630759e8/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E5%85%83%E7%B4%A0%E6%9F%A5%E6%89%BE.md#pluck)》

否则，直接返回第一个元素的 `value` 值。

## .offsetParent()

```javascript
ootNodeRE = /^(?:body|html)$/i
offsetParent: function() {
  return this.map(function() {
    var parent = this.offsetParent || document.body
    while (parent && !rootNodeRE.test(parent.nodeName) && $(parent).css("position") == "static")
      parent = parent.offsetParent
    return parent
  })
}
```

查找最近的祖先定位元素，即最近的属性 `position` 被设置为 `relative` 、`absolute` 和 `fixed` 的祖先元素。

```javascript
var parent = this.offsetParent || document.body
```

获取元素的 `offsetParent` 属性，如果不存在，则默认赋值为 `body` 元素。

```javascript
parent && !rootNodeRE.test(parent.nodeName) && $(parent).css("position") == "static"
```

判断父级定位元素是否存在，并且不为根元素（即 `body` 元素或 `html` 元素），并且为相对定位元素，才进入循环，循环内是获取下一个 `offsetParent` 元素。

这个应该做浏览器兼容的吧，因为 `offsetParent` 本来返回的就是最近的定位元素。

## .offset()

```javascript
offset: function(coordinates) {
  if (coordinates) return this.each(function(index) {
    var $this = $(this),
        coords = funcArg(this, coordinates, index, $this.offset()),
        parentOffset = $this.offsetParent().offset(),
        props = {
          top: coords.top - parentOffset.top,
          left: coords.left - parentOffset.left
        }

    if ($this.css('position') == 'static') props['position'] = 'relative'
    $this.css(props)
  })
  if (!this.length) return null
  if (document.documentElement !== this[0] && !$.contains(document.documentElement, this[0]))
    return { top: 0, left: 0 }
    var obj = this[0].getBoundingClientRect()
    return {
      left: obj.left + window.pageXOffset,
      top: obj.top + window.pageYOffset,
      width: Math.round(obj.width),
      height: Math.round(obj.height)
    }
},
```

获取或设置元素相对 `document` 的偏移量。

先来看获取值：

```javascript
if (!this.length) return null
if (document.documentElement !== this[0] && !$.contains(document.documentElement, this[0]))
  return { top: 0, left: 0 }
var obj = this[0].getBoundingClientRect()
return {
  left: obj.left + window.pageXOffset,
  top: obj.top + window.pageYOffset,
  width: Math.round(obj.width),
  height: Math.round(obj.height)
}
```

如果集合不存在，则返回 `null`

```javascript
if (document.documentElement !== this[0] && !$.contains(document.documentElement, this[0]))
  return { top: 0, left: 0 }
```

如果集合中第一个元素不为 `html` 元素对象(`document.documentElement !== this[0]`) ，并且不为 `html` 元素的子元素，则返回 `{ top: 0, left: 0 }`

接下来，调用 `getBoundingClientRect` ，获取元素的 `width` 和 `height` 值，以及相对视窗左上角的 `left` 和 `top` 值。具体参见文档: [Element.getBoundingClientRect()](https://developer.mozilla.org/en-US/docs/Web/API/Element/getBoundingClientRect)

因为 `getBoundingClientRect` 获取到的位置是相对视窗的，因此需要将视窗外偏移量加上，即加上 `window.pageXOffset` 或 `window.pageYOffset` 。

再来看设置值：

```javascript
if (coordinates) return this.each(function(index) {
  var $this = $(this),
      coords = funcArg(this, coordinates, index, $this.offset()),
      parentOffset = $this.offsetParent().offset(),
      props = {
        top: coords.top - parentOffset.top,
        left: coords.left - parentOffset.left
      }

  if ($this.css('position') == 'static') props['position'] = 'relative'
  $this.css(props)
})
```

前面几行都是固有的模式，不再展开，看看这段：

```javascript
parentOffset = $this.offsetParent().offset()
```

获取最近定位元素的 `offset` 值，这个值有什么用呢？

```javascript
props = {
  top: coords.top - parentOffset.top,
  left: coords.left - parentOffset.left
}
if ($this.css('position') == 'static') props['position'] = 'relative'
  $this.css(props)
```

我们可以看到，设置偏移的时候，其实是设置元素的 `left` 和 `top` 值。如果父级元素有定位元素，那这个 `left` 和 `top` 值是相对于第一个父级定位元素的。

因此需要将传入的 `coords.top` 和 `coords.left` 对应减掉第一个父级定位元素的 `offset` 的 `top` 和 `left` 值。

如果当前元素的 `position` 值为 `static` ，则将值设置为 `relative` ，相对自身偏移计算出来相差的 `left` 和 `top` 值。

## .position()

```javascript
position: function() {
  if (!this.length) return

  var elem = this[0],
    offsetParent = this.offsetParent(),
    offset = this.offset(),
    parentOffset = rootNodeRE.test(offsetParent[0].nodeName) ? { top: 0, left: 0 } : offsetParent.offset()
  offset.top -= parseFloat($(elem).css('margin-top')) || 0
  offset.left -= parseFloat($(elem).css('margin-left')) || 0
  parentOffset.top += parseFloat($(offsetParent[0]).css('border-top-width')) || 0
  parentOffset.left += parseFloat($(offsetParent[0]).css('border-left-width')) || 0
  return {
    top: offset.top - parentOffset.top,
    left: offset.left - parentOffset.left
  }
},
```

返回相对父元素的偏移量。

```javascript
offsetParent = this.offsetParent(),
offset = this.offset(),
parentOffset = rootNodeRE.test(offsetParent[0].nodeName) ? { top: 0, left: 0 } : offsetParent.offset()
```

分别获取到第一个定位父元素 `offsetParent` 及相对文档偏移量 `parentOffset` ，和自身的相对文档偏移量 `offset` 。在获取每一个定位父元素偏移量时，先判断父元素是否为根元素，如果是，则 `left` 和 `top` 都返回 `0`。

```javascript
offset.top -= parseFloat($(elem).css('margin-top')) || 0
offset.left -= parseFloat($(elem).css('margin-left')) || 0
```

两个元素之间的距离应该不包含元素的外边距，因此将外边距减去。

```javascript
parentOffset.top += parseFloat($(offsetParent[0]).css('border-top-width')) || 0
parentOffset.left += parseFloat($(offsetParent[0]).css('border-left-width')) || 0
```

因为 `position` 返回的是距离第一个定位元素的 `context box` 的距离，因此父元素的 `offset` 的 `left` 和 `top` 值需要将 `border` 值加上（`offset` 算是的外边距距离文档的距离）。

```javascript
return {
  top: offset.top - parentOffset.top,
  left: offset.left - parentOffset.left
}
```

最后，将他们距离文档的偏移量相减就得到两者间的偏移量了。

## .scrollTop()

```javascript
scrollTop: function(value) {
  if (!this.length) return
  var hasScrollTop = 'scrollTop' in this[0]
  if (value === undefined) return hasScrollTop ? this[0].scrollTop : this[0].pageYOffset
  return this.each(hasScrollTop ?
                   function() { this.scrollTop = value } :
                   function() { this.scrollTo(this.scrollX, value) })
},
```

获取或设置元素在纵轴上的滚动距离。

先看获取值：

```javascript
var hasScrollTop = 'scrollTop' in this[0]
if (value === undefined) return hasScrollTop ? this[0].scrollTop : this[0].pageYOffset
```

如果存在 `scrollTop` 属性，则直接用 `scrollTop` 获取属性，否则用 `pageYOffset` 获取元素Y轴在屏幕外的距离，也即滚动高度了。

```javascript
return this.each(hasScrollTop ?
                   function() { this.scrollTop = value } :
                   function() { this.scrollTo(this.scrollX, value) })
```

知道了获取值后，设置值也简单了，如果有 `scrollTop` 属性，则直接设置这个属性的值，否则调用 `scrollTo` 方法，用 `scrollX` 获取到 `x` 轴的滚动距离，将 `y` 轴滚动到指定的距离 `value`。

## .scrollLeft()

```javascript
scrollLeft: function(value) {
  if (!this.length) return
  var hasScrollLeft = 'scrollLeft' in this[0]
  if (value === undefined) return hasScrollLeft ? this[0].scrollLeft : this[0].pageXOffset
  return this.each(hasScrollLeft ?
                   function() { this.scrollLeft = value } :
                   function() { this.scrollTo(value, this.scrollY) })
},
```

`scrollLeft` 原理同 `scrollTop` ，不再展开叙述。

## 系列文章

1. [读Zepto源码之代码结构](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E4%BB%A3%E7%A0%81%E7%BB%93%E6%9E%84.md)
2. [读 Zepto 源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md)
3. [读Zepto源码之工具函数](https://github.com/yeyuqiudeng/reading-zepto/blob/a4d6ad99c57047beae2b652b4d2cbb380599a524/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md)
4. [读Zepto源码之神奇的$](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E7%A5%9E%E5%A5%87%E7%9A%84%24.md)
5. [读Zepto源码之集合操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E6%93%8D%E4%BD%9C.md)
6. [读Zepto源码之集合元素查找](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E5%85%83%E7%B4%A0%E6%9F%A5%E6%89%BE.md)
7. [读Zepto源码之操作DOM](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E6%93%8D%E4%BD%9CDOM.md)
8. [读Zepto源码之样式操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E6%A0%B7%E5%BC%8F%E6%93%8D%E4%BD%9C.md)

## 参考

* [MDN:Node.textContent](https://developer.mozilla.org/en-US/docs/Web/API/Node/textContent)

* [data-*](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/data-*)

* [Element.getBoundingClientRect()](https://developer.mozilla.org/en-US/docs/Web/API/Element/getBoundingClientRect)

  ​

## License

[![License: CC BY-NC-ND 4.0](https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://user-gold-cdn.xitu.io/2017/5/30/76626b0be42083d36b36f4a117dc1873) 

作者：对角另一面