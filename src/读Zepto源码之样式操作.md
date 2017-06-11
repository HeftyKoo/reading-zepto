# 读Zepto源码之样式操作

这篇依然是跟 `dom` 相关的方法，侧重点是操作样式的方法。

读Zepto源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## 内部方法

### classRE

```javascript
classCache = {}

function classRE(name) {
  return name in classCache ?
    classCache[name] : (classCache[name] = new RegExp('(^|\\s)' + name + '(\\s|$)'))
}
```

这个函数是用来返回一个正则表达式，这个正则表达式是用来匹配元素的 `class` 名的，匹配的是如 `className1 className2 className3` 这样的字符串。

`calssCache` 初始化时是一个空对象，用 `name` 用为 `key` ，如果正则已经生成过，则直接从 `classCache` 中取出对应的正则表达式。

否则，生成一个正则表达式，存储到 `classCache` 中，并返回。

来看一下这个生成的正则，`'(^|\\s)'` 匹配的是开头或者空白（包括空格、换行、tab缩进等），然后连接指定的 `name` ，再紧跟着空白或者结束。

### maybeAddPx

```javascript
cssNumber = { 'column-count': 1, 'columns': 1, 'font-weight': 1, 'line-height': 1, 'opacity': 1, 'z-index': 1, 'zoom': 1 }

function maybeAddPx(name, value) {
  return (typeof value == "number" && !cssNumber[dasherize(name)]) ? value + "px" : value
}
```

在给属性设置值时，猜测所设置的属性可能需要带 `px` 单位时，自动给值拼接上单位。

`cssNumber` 是不需要设置 `px` 的属性值，所以这个函数里首先判断设置的值是否为 `number` 类型，如果是，并且需要设置的属性不在 `cssNumber` 中时，给值拼接上 `px` 单位。

### defaultDisplay

```javascript
elementDisplay = {}

function defaultDisplay(nodeName) {
  var element, display
  if (!elementDisplay[nodeName]) {
    element = document.createElement(nodeName)
    document.body.appendChild(element)
    display = getComputedStyle(element, '').getPropertyValue("display")
    element.parentNode.removeChild(element)
    display == "none" && (display = "block")
    elementDisplay[nodeName] = display
  }
  return elementDisplay[nodeName]
}
```

先透露一下，这个方法是给 `.show()` 用的，`show` 方法需要将元素显示出来，但是要显示的时候能不能直接将 `display` 设置成 `block` 呢？显然是不行的，来看一下 `display` 的可能会有那些值：

```javascript
display: none

display: inline
display: block
display: contents
display: list-item
display: inline-block
display: inline-table
display: table
display: table-cell
display: table-column
display: table-column-group
display: table-footer-group
display: table-header-group
display: table-row
display: table-row-group
display: flex
display: inline-flex
display: grid
display: inline-grid
display: ruby
display: ruby-base
display: ruby-text
display: ruby-base-container
display: ruby-text-container 
display: run-in

display: inherit
display: initial
display: unset
```

如果元素原来的 `display` 值为 `table` ，调用 `show` 后变成 `block` 了，那页面的结构可能就乱了。

这个方法就是将元素显示时默认的 `display` 值缓存到 `elementDisplay`，并返回。

函数用节点名 `nodeName` 为 `key` ，如果该节点显示时的 `display` 值已经存在，则直接返回。

```javascript
element = document.createElement(nodeName)
document.body.appendChild(element)
```

否则，使用节点名创建一个空元素，并且将元素插入到页面中

```javascript
display = getComputedStyle(element, '').getPropertyValue("display")
element.parentNode.removeChild(element)
```

调用 `getComputedStyle` 方法，获取到元素显示时的 `display` 值。获取到值后将所创建的元素删除。

```javascript
display == "none" && (display = "block")
elementDisplay[nodeName] = display
```

如果获取到的 `display` 值为 `none` ，则将显示时元素的 `display` 值默认为 `block`。然后将结果缓存起来。`display` 的默认值为 `none`？ Are you kiding me ? 真的有这种元素吗？还真的有，像 `style`、 `head` 和 `title` 等元素的默认值都是 `none` 。将 `style` 和 `head` 的 `display`  设置为 `block` ，并且将 `style` 的 `contenteditable` 属性设置为 `true` ，`style` 就显示出来了，直接在页面上一边敲样式，一边看效果，爽！！！

关于元素的 `display` 默认值，可以看看这篇文章 [Default CSS Display Values for Different HTML Elements](https://www.impressivewebs.com/default-css-display-values-html-elements/)

### funcArg

```javascript
function funcArg(context, arg, idx, payload) {
  return isFunction(arg) ? arg.call(context, idx, payload) : arg
}
```

这个函数要注意，本篇和下一篇介绍的绝大多数方法都会用到这个函数。

例如本篇将要说到的 `addClass` 和 `removeClass` 等方法的参数可以为固定值或者函数，这些方法的参数即为形参 `arg`。

当参数 `arg` 为函数时，调用 `arg` 的 `call` 方法，将上下文 `context` ，当前元素的索引 `idx` 和原始值 `payload` 作为参数传递进去，将调用结果返回。

如果为固定值，直接返回 `arg`

### className

```javascript
function className(node, value) {
  var klass = node.className || '',
      svg = klass && klass.baseVal !== undefined

  if (value === undefined) return svg ? klass.baseVal : klass
  svg ? (klass.baseVal = value) : (node.className = value)
}
```

`className` 包含两个参数，为元素节点 `node` 和需要设置的样式名 `value`。

如果 `value` 不为 `undefined`（可以为空，注意判断条件为 `value === undefined`，用了全等判断），则将元素的 `className` 设置为给定的值，否则将元素的 `className` 值返回。

这个函数对 `svg` 的元素做了兼容，如果元素的 `className` 属性存在，并且 `className` 属性存在 `baseVal` 时，为 `svg` 元素，如果是 `svg` 元素，取值和赋值都是通过 `baseVal` 。对 `svg` 不是很熟，具体见文档: [SVGAnimatedString.baseVal](https://developer.mozilla.org/en-US/docs/Web/API/SVGAnimatedString/baseVal)

## .css()

```javascript
css: function(property, value) {
  if (arguments.length < 2) {
    var element = this[0]
    if (typeof property == 'string') {
      if (!element) return
      return element.style[camelize(property)] || getComputedStyle(element, '').getPropertyValue(property)
        } else if (isArray(property)) {
          if (!element) return
          var props = {}
          var computedStyle = getComputedStyle(element, '')
          $.each(property, function(_, prop) {
            props[prop] = (element.style[camelize(prop)] || computedStyle.getPropertyValue(prop))
          })
          return props
        }
  }

  var css = ''
  if (type(property) == 'string') {
    if (!value && value !== 0)
      this.each(function() { this.style.removeProperty(dasherize(property)) })
      else
        css = dasherize(property) + ":" + maybeAddPx(property, value)
        } else {
          for (key in property)
            if (!property[key] && property[key] !== 0)
              this.each(function() { this.style.removeProperty(dasherize(key)) })
              else
                css += dasherize(key) + ':' + maybeAddPx(key, property[key]) + ';'
                }

  return this.each(function() { this.style.cssText += ';' + css })
}
```

`css` 方法有两个参数，`property` 是的 `css` 样式名，`value` 是需要设置的值，如果不传递 `value` 值则为取值操作，否则为赋值操作。

来看看调用方式：

```javascript
css(property)   ⇒ value  // 获取值
css([property1, property2, ...])   ⇒ object // 获取值
css(property, value)   ⇒ self // 设置值
css({ property: value, property2: value2, ... })   ⇒ self // 设置值
```

 下面这段便是处理获取值情况的代码：

```javascript
if (arguments.length < 2) {
  var element = this[0]
  if (typeof property == 'string') {
    if (!element) return
    return element.style[camelize(property)] || getComputedStyle(element, '').getPropertyValue(property)
      } else if (isArray(property)) {
        if (!element) return
        var props = {}
        var computedStyle = getComputedStyle(element, '')
        $.each(property, function(_, prop) {
          props[prop] = (element.style[camelize(prop)] || computedStyle.getPropertyValue(prop))
        })
        return props
      }
}
```

当为获取值时，`css` 方法必定只传递了一个参数，所以用 `arguments.length < 2` 来判断，用 `css` 方法来获取值，获取的是集合中第一个元素对应的样式值。

```javascript
if (!element) return
return element.style[camelize(property)] || getComputedStyle(element, '').getPropertyValue(property)
```

当 `property` 为 `string` 时，如果元素不存在，直接 `return` 掉。

如果 `style` 中存在对应的样式值，则优先获取 `style` 中的样式值，否则用 `getComputedStyle` 获取计算后的样式值。

为什么不直接获取计算后的样式值呢？因为用 `style` 获取的样式值是原始的字符串，而 `getComputedStyle` 顾名思义获取到的是计算后的样式值，如 `style = "transform: translate(10px, 10px)"` 用 `style.transform` 获取到的值为 `translate(10px, 10px)`，而用 `getComputedStyle` 获取到的是 `matrix(1, 0, 0, 1, 10, 10)`。这里用到的 `camelize` 方法是将属性 `property` 转换成驼峰式的写法，该方法在《[读Zepto源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md#camelize)》有过分析。

```javascript
else if (isArray(property)) {
  if (!element) return
  var props = {}
  var computedStyle = getComputedStyle(element, '')
  $.each(property, function(_, prop) {
    props[prop] = (element.style[camelize(prop)] || computedStyle.getPropertyValue(prop))
  })
  return props
}
```

如果参数 `property` 为数组时，表示要获取一组属性的值。`isArray` 方法也在《[读Zepto源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md#isarray)》有过分析。

获取的方法也很简单，遍历 `property` ，获取 `style` 上对应的样式值，如果 `style` 上的值不存在，则通过 `getComputedStyle` 来获取，返回的是以样式名为 `key` ，`value` 为对应的样式值的对象。

接下来是给所有元素设置值的情况：

```javascript
var css = ''
if (type(property) == 'string') {
  if (!value && value !== 0)
    this.each(function() { this.style.removeProperty(dasherize(property)) })
  else
    css = dasherize(property) + ":" + maybeAddPx(property, value)
 } else {
    for (key in property)
        if (!property[key] && property[key] !== 0)
            this.each(function() { this.style.removeProperty(dasherize(key)) })
         else
            css += dasherize(key) + ':' + maybeAddPx(key, property[key]) + ';'
         }

return this.each(function() { this.style.cssText += ';' + css })
```

这里定义了个变量 `css` 来接收需要新值的样式字符串。

```javascript
if (type(property) == 'string') {
  if (!value && value !== 0)
    this.each(function() { this.style.removeProperty(dasherize(property)) })
  else
    css = dasherize(property) + ":" + maybeAddPx(property, value)
 }
```

当参数 `property` 为字符串时

如果 `value` 不存在并且值不为 `0` 时（注意，`value` 为 `undefined` 时，已经在上面处理过了，也即是获取样式值），遍历集合，将对应的样式值从 `style` 中删除。

否则，拼接样式字符串，拼接成如 `width:100px` 形式的字符串。这里调用了 `maybeAddPx` 的方法，自动给需要加 `px` 的属性值拼接上了 `px` 单位。`this.css('width', 100)` 跟 `this.css('width', '100px')` 会得到一样的结果。

```javascript
for (key in property)
  if (!property[key] && property[key] !== 0)
    this.each(function() { this.style.removeProperty(dasherize(key)) })
    else
      css += dasherize(key) + ':' + maybeAddPx(key, property[key]) + ';'
```

当 `property` 为 `key` 是样式名，`value` 为样式值的对象时，用 `for...in` 遍历对象，接下来的处理逻辑跟 `property` 为 `string` 时差不多，在做 `css` 拼接时，在末尾加了 `;`，避免遍历时，将样式名和值连接在了一起。

## .hide()

```javascript
hide: function() {
  return this.css("display", "none")
},
```

将集合中所有元素的 `display` 样式属性设置为 `node`，就达到了隐藏元素的目的。注意，`css` 方法中已经包含了 `each` 循环。

## .show()

```javascript
show: function() {
  return this.each(function() {
    this.style.display == "none" && (this.style.display = '')
    if (getComputedStyle(this, '').getPropertyValue("display") == "none")
      this.style.display = defaultDisplay(this.nodeName)
      })
},
```

`hide` 方法是直接将 `display` 设置为 `none` 即可，`show` 可不可以直接将需要显示的元素的 `display` 设置为 `block` 呢？

这样在大多数情况下是可以的，但是碰到像 `table` 、`li` 等显示时 `display` 默认值不是 `block` 的元素，强硬将它们的 `display` 属性设置为 `block` ，可能会更改他们的默认行为。

`show` 要让元素真正显示，要经过两步检测：

```javascript
this.style.display == "none" && (this.style.display = '')
```

如果 `style` 中的 `display` 属性为 `none` ，先将 `style` 中的 `display` 置为 ``。

```javascript
if (getComputedStyle(this, '').getPropertyValue("display") == "none")
  this.style.display = defaultDisplay(this.nodeName)
 })
```

这样还未完，内联样式的 `display` 属性是置为空了，但是如果嵌入样式或者外部样式表中设置了 `display` 为 `none` 的样式，或者本身的 `display` 默认值就是 `none` 的元素依然显示不了。所以还需要用获取元素的计算样式，如果为 `none` ，则将 `display` 的属性设置为元素显示时的默认值。如 `table` 元素的 `style` 中的 `display` 属性值会被设置为 `table`。

## .toggle()

```javascript
toggle: function(setting) {
  return this.each(function() {
    var el = $(this);
    (setting === undefined ? el.css("display") == "none" : setting) ? el.show(): el.hide()
  })
},
```

切换元素的显示和隐藏状态，如果元素隐藏，则显示元素，如果元素显示，则隐藏元素。可以用参数 `setting` 指定 `toggle` 的行为，如果指定为 `true` ，则显示，如果为 `false` （ `setting` 不一定为 `Boolean`），则隐藏。

注意，判断条件是 `setting === undefined` ，用了全等，只有在不传参，或者传参为 `undefined` 的时候，条件才会成立。

## .hasClass()

```javascript
hasClass: function(name) {
  if (!name) return false
  return emptyArray.some.call(this, function(el) {
    return this.test(className(el))
  }, classRE(name))
},
```

判断集合中的元素是否存在指定 `name` 的 `class` 名。

如果没有指定 `name` 参数，则直接返回 `false`。

否则，调用 `classRE` 方法，生成检测样式名的正则，传入数组方法 `some`，要注意， `some` 里面的 `this` 值并不是遍历的当前元素，而是传进去的 `classRE(name)` 正则，回调函数中的 `el` 才是当前元素。具体参考文档 [Array.prototype.some()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/some?v=example)

调用 `className` 方法，获取当前元素的 `className` 值，如果有一个元素匹配了正则，则返回 `true`。

## .addClass()

```javascript
addClass: function(name) {
  if (!name) return this
  return this.each(function(idx) {
    if (!('className' in this)) return
    classList = []
    var cls = className(this),
        newName = funcArg(this, name, idx, cls)
    newName.split(/\s+/g).forEach(function(klass) {
      if (!$(this).hasClass(klass)) classList.push(klass)
        }, this)
    classList.length && className(this, cls + (cls ? " " : "") + classList.join(" "))
  })
},
```

为集合中的所有元素增加指定类名 `name`。 `name` 可以为固定值或者函数。

如果 `name` 没有传递，则返回当前集合 `this` ，以进行链式操作。

如果 `name` 存在，遍历集合，判断当前元素是否存在 `className` 属性，如果不存在，立即退出循环。要注意，在 `each` 遍历中，`this` 指向的是当前元素。

```javascript
classList = []
var cls = className(this),
    newName = funcArg(this, name, idx, cls)
```

`classList` 用来接收需要增加的样式类数组。不太明白为什么要用全局变量 `classList` 来接收，用局部变量不是更好点吗？

`cls`  保存当前类的字符串，使用函数 `className` 获得。

`newName` 是需要新增的样式类字符串，因为 `name` 可以是函数或固定值，统一交由 `funcArg` 来处理。

```javascript
newName.split(/\s+/g).forEach(function(klass) {
  if (!$(this).hasClass(klass)) classList.push(klass)
    }, this)
classList.length && className(this, cls + (cls ? " " : "") + classList.join(" "))
```

`newName.split(/\s+/g)` 是将 `newName` 字符串，用空白分割成数组。

再对数组遍历，得到单个类名，调用 `hasClass` 判断类名是否已经存在于元素的 `className` 中，如果不存在，将类名 `push` 进数组 `classList` 中。

如果 `classList` 不为空，则调用 `className` 方法给元素设置值。`classList.join(" ")` 是将类名转换成用空格分隔的字符串，如果 `cls` 即元素原来就存在有其他类名，拼接时也使用空格分隔开。

## .removeClass()

```javascript
removeClass: function(name) {
  return this.each(function(idx) {
    if (!('className' in this)) return
    if (name === undefined) return className(this, '')
    classList = className(this)
    funcArg(this, name, idx, classList).split(/\s+/g).forEach(function(klass) {
      classList = classList.replace(classRE(klass), " ")
    })
    className(this, classList.trim())
  })
},
```

删除元素中指定的类 `name` 。如果不传递参数，则将 `className` 属性置为空，也即删除所有样式类。

```javascript
classList = className(this)
funcArg(this, name, idx, classList).split(/\s+/g).forEach(function(klass) {
  classList = classList.replace(classRE(klass), " ")
})
className(this, classList.trim())
```

这是的 `classList` 依然是全局变量，但是接收的是当前元素的当前样式类字符串（为什么不用局部变量呢？）。

参数 `name` 依然可以为函数或者固定值，因此用 `funcArg` 来处理，然后用空白分割成数组，再遍历得到单个样式类，调用 `replace` 方法，如果 `classList` 中能匹配到这个类，则将匹配的字符串替换成空格，这样就达到了删除的目的。

 最后，用 `trim` 将 `classList` 的头尾空格去掉，调用 `className` 方法，重新给当前元素的 `className` 赋值。

## .toggleClass()

```javascript
toggleClass: function(name, when) {
  if (!name) return this
  return this.each(function(idx) {
    var $this = $(this),
        names = funcArg(this, name, idx, className(this))
    names.split(/\s+/g).forEach(function(klass) {
      (when === undefined ? !$this.hasClass(klass) : when) ?
        $this.addClass(klass): $this.removeClass(klass)
    })
  })
},
```

切换样式类，如果样式类不存在，则增加样式类，如果存在，则删除样式类。

`toggleClass` 接收两个参数，`name` 是需要切换的类名， `when` 是指定切换的方法，如果 `when` 为 `true` ，则增加样式类，为 `false` ，则删除样式类。`when` 不一定要为 `Boolean` 类型。

这个方法跟 `toggle` 方法的逻辑参不多，只不过调用的方法变成 `addClass` 和 `removeClass` ，可以参考 `toggle` 的实现，不用过多分析。

## 系列文章

1. [读Zepto源码之代码结构](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E4%BB%A3%E7%A0%81%E7%BB%93%E6%9E%84.md)
2. [读 Zepto 源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md)
3. [读Zepto源码之工具函数](https://github.com/yeyuqiudeng/reading-zepto/blob/a4d6ad99c57047beae2b652b4d2cbb380599a524/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md)
4. [读Zepto源码之神奇的$](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E7%A5%9E%E5%A5%87%E7%9A%84%24.md)
5. [读Zepto源码之集合操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E6%93%8D%E4%BD%9C.md)
6. [读Zepto源码之集合元素查找](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E5%85%83%E7%B4%A0%E6%9F%A5%E6%89%BE.md)
7. [读Zepto源码之操作DOM](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E6%93%8D%E4%BD%9CDOM.md)

## 参考

* [MDN: display](https://developer.mozilla.org/en-US/docs/Web/CSS/display?v=control)
* [Default CSS Display Values for Different HTML Elements](https://www.impressivewebs.com/default-css-display-values-html-elements/)
* [SVGAnimatedString.baseVal](https://developer.mozilla.org/en-US/docs/Web/API/SVGAnimatedString/baseVal)
* [获取元素CSS值之getComputedStyle方法熟悉](http://www.zhangxinxu.com/wordpress/2012/05/getcomputedstyle-js-getpropertyvalue-currentstyle/)
* [Array.prototype.some()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/some?v=example)

## License

[![License: CC BY-NC-ND 4.0](https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://user-gold-cdn.xitu.io/2017/5/30/76626b0be42083d36b36f4a117dc1873) 

作者：对角另一面