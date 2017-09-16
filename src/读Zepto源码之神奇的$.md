# 读Zepto源码之神奇的$  

经过前面三章的铺垫，这篇终于写到了戏肉。在用 `zepto` 时，肯定离不开这个神奇的 `$` 符号，这篇文章将会看看 `zepto` 是如何实现 `$` 的。

读Zepto源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## zepto的css选择器 `zepto.qsa`

我们都知道，很多时候，我们都用`$` 来获取DOM对象，这跟 `zepto.qsa` 有很大的关系。 

### 源码

```javascript
zepto.qsa = function(element, selector) {
        var found,  // 已经找的到DOM
            maybeID = selector[0] == '#',  // 是否为ID
            maybeClass = !maybeID && selector[0] == '.', // 是否为class
            nameOnly = maybeID || maybeClass ? selector.slice(1) : selector,  // 将id或class前面的符号去掉
            isSimple = simpleSelectorRE.test(nameOnly)  // 是否为单个选择器
        return (element.getElementById && isSimple && maybeID) ? 
            ((found = element.getElementById(nameOnly)) ? [found] : []) :
            (element.nodeType !== 1 && element.nodeType !== 9 && element.nodeType !== 11) ? [] :
            slice.call(
                isSimple && !maybeID && element.getElementsByClassName ? 
                maybeClass ? element.getElementsByClassName(nameOnly) : 
                element.getElementsByTagName(selector) : 
                element.querySelectorAll(selector) 
            )
    }
```

以上是 `qsa` 的所有代码，里面有用到一个正则表达式 `simpleSelectorRE`，先将这个正则消化下。

```javascript
simpleSelectorRE = /^[\w-]*$/,
```

看到这个正则其实是匹配 `a-z、A-Z、0-9、下划线、连词符` 组合起来的单词，这其实就是单个 `id` 和 `class` 的命名规则。

从 `return` 中可以看出，`qsa` 其实是根据不同情况分别调用了原生的 `getElementById`、`getElementsByClassName` 、`getElementsByTagName` 和 `querySelectorAll` 的方法。

为什么要这么麻烦，不直接调用 `querySelectorAll` 方法呢？这是出于性能的考虑。这里有个简单的[测试](http://jsbin.com/domucixawu/edit?html,css,js,console,output)。这个测试里，页面上只有一个元素，如果比较复杂的时候，差距更加明显。

好了，开始逐行分析代码。

### 参数

* element 开始查找的元素
* selector 选择器

###  变量

* `found`： 已经找到的元素
* `maybeID = selector[0] == '#'`： 判断选择器的第一个字符是否为 `#`， 如果是 `# ` ，则可能是 `id` 选择器
* `maybeClass = !maybeID && selector[0] == '.'` 如果不是 `id` 选择器，并且选择器的第一个字符为 `.` ，则可能是 `class` 选择器
* `nameOnly = maybeID || maybeClass ? selector.slice(1) : selector` ，如果为 `id` 选择器或者 `class` 选择器，则将第一个字符去掉
* `isSimple = simpleSelectorRE.test(nameOnly)` 是否为单选择器，即 `.single` 的形式，不是 `.first .secend` 等形式

###  element.getElementById

`(element.getElementById && isSimple && maybeID)` 这是采用  `element.getElementById` 的条件。

首先要确保 `element` 具有 `getElementById` 的方法。`getElementById` 的方法是在 `document` 上的，Chrome等浏览器上，`element` 可能并不具有 `geElementById` 的方法，具体可以看看这篇文章：[各浏览器对document.getElementById等方法的实现差异解析](http://www.jb51.net/article/44147.htm)

然后要确保选择器为单选择器，并且为 `id` 选择器。

返回值为 `((found = element.getElementById(nameOnly)) ? [found] : [])`， 如果能查找到元素，则将元素以数组的形式返回，否则返回空数组

### 排除不合法的element

`element.nodeType !== 1 && element.nodeType !== 9 && element.nodeType !== 11` 。`1` 对应的是 `Node.ELEMENT_NODE` ，`10` 对应的是 `Node.DOCUMENT_TYPE_NODE` ， `11` 对应的是 `Node.DOCUMENT_FRAGMENT_NODE` ，如果不为以上三种类型，直接返回 `[]`。 

###  终极三元表达式

```javascript
slice.call(
  isSimple && !maybeID && element.getElementsByClassName ?  // 如果为单选择器并且不为id选择器并且存在getElementsByClassName方法，进入下一个三元表达式判断
  maybeClass ? element.getElementsByClassName(nameOnly) :   // 如果为class选择器，则采用getElementsByClassName
  element.getElementsByTagName(selector) :  // 否则采用getElementsByTagName方法
  element.querySelectorAll(selector)   // 以上情况都不是，则用querySelectorAll
)
```

这里用了 `slice.call` 处理所获取到的集合，这样，获取到的DOM集合就可以直接使用数组的方法了。

## zepto.Z 函数

从第一篇[代码结构](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E4%BB%A3%E7%A0%81%E7%BB%93%E6%9E%84.md)中我们已经知道，其实实现 `$` 函数的核心是 `zepto.init` ，而 `zepto.init` 最终返回的是 `zepto.Z` 的结果。那就先来看看 `zepto.Z`

```javascript
zepto.Z = function(dom, selector) {
  return new Z(dom, selector)
}
```

`zepto.Z` 的代码很简单，返回的是 `Z` 函数的实例。那接下来再看看 `Z` 函数：

```javascript
function Z(dom, selector) {
  var i, len = dom ? dom.length : 0
  for (i = 0; i < len; i++) this[i] = dom[i]
  this.length = len
  this.selector = selector || ''
}
```

`Z` 函数做的事情也很简单，就是将 `dom` 数组转化为类数组的形式，并设置对应的 `length` 属性和 `selector` 属性。

### zepto.isZ 

```javascript
zepto.isZ = function(object) {
  return object instanceof zepto.Z
}
```

既然看了 `Z` 函数，就顺便也将 `isZ` 也一起看了吧。`isZ` 函数用来判断参数 `object` 是否为 `Z` 的实例，这在 `init` 中会用到。

## $的实现 zepto.init 函数

### $的实现

```javascript
$ = function(selector, context) {
  return zepto.init(selector, context)
}
```

可以看到，其实 `$` 调用的就是 `zepto.init` 这个内部方法。

### zepto.init

```javascript
zepto.init = function(selector, context) {
  var dom  // dom 集合
  if (!selector) return zepto.Z() // 分支1
  else if (typeof selector == 'string') { // 分支2
    selector = selector.trim()
    if (selector[0] == '<' && fragmentRE.test(selector))
      dom = zepto.fragment(selector, RegExp.$1, context), selector = null
      else if (context !== undefined) return $(context).find(selector)
      else dom = zepto.qsa(document, selector)
        }
  else if (isFunction(selector)) return $(document).ready(selector) // 分支3
  else if (zepto.isZ(selector)) return selector  // 分支4
  else { // 分支5
    if (isArray(selector)) dom = compact(selector)
    else if (isObject(selector))
      dom = [selector], selector = null
      else if (fragmentRE.test(selector))
        dom = zepto.fragment(selector.trim(), RegExp.$1, context), selector = null
        else if (context !== undefined) return $(context).find(selector)
        else dom = zepto.qsa(document, selector)
          }
  return zepto.Z(dom, selector)
}
```

 这个 `init` 方法代码量不多，但是有大量的 `if else`， 希望我可以说得清楚

###  $的用法

```javascript
$(selector, [context])   ⇒ collection  // 用法1
$(<Zepto collection>)   ⇒ same collection // 用法2
$(<DOM nodes>)   ⇒ collection // 用法3
$(htmlString)   ⇒ collection // 用法4
$(htmlString, attributes)   ⇒ collection v1.0+ // 用法5
Zepto(function($){ ... })   // 用法6
```

### 不传参调用

直接调用 `$()` 时，对应的是**分支1**的情况： `if (!selector) return zepto.Z()` ，返回的是空的 `Z` 对象

### `selector` 为 `String` 时

当 `selector` 为 `string` 时，对应的代码在**分支2**，对应的用法是**用法1**、**用法4**和**用法5**

在这个分支里，又有三个子分支。一一来看一下：

第一个的判断条件为 `selector[0] == '<' && fragmentRE.test(selector)` 。`selector` 的第一个字符为 `<` ，并且为html标签 。`fragmentRE` 的定义如下 `fragmentRE = /^\s*<(\w+|!)[^>]*>/` ，这个其实就是用来判断字符串是否为标签。 我对正则也不太熟，这里就不再展开。

如果满足条件，则执行如下代码：`dom = zepto.fragment(selector, RegExp.$1, context), selector = null`。 `zepto.fragment` 其实是通过 `htmlString` 返回一个dom集合。这个函数稍后会说到，这里先不展开。这里对应的是**用法4**和**用法5**。

如果不满足第一个判断条件，则再判断 `context !== undefined` (上下文是否存在)。如果存在，则查找 `context` 下选择器为 `selector` 的所有子元素: ` $(context).find(selector)` 。这个分支对应的是**用法1**

否则，调用 `zepto.qsa` 方法，查找 `document` 下的所有 `selector` ： `dom = zepto.qsa(document, selector)`。这里对应的是**用法1**。

### `selector` 为 `Function` 时

对应的代码在**分支3**，对应的用法是**用法6**

这个分支很简单，在页面加载完毕后，再执行回调方法：`$(document).ready(selector)`

用过 `zepto` 的应该都熟悉这种用法: `$(function() {})`。其实走的就是这个分支

### `selector` 为 `Z` 对象时

对应的代码在**分支4**，对应的用法是**用法2**

如果参数已经为 `Z` 对象（`zepto.isZ(selector)`），则不需要做任何事情，直接原对象返回就可以了。

### `selector` 为其他情况

如果为数组时（`isArray(selector)`）, 将数组展平(`dom = compact(selector)`)

如果为对象时（`isObject(selector)`），将对象包裹成数组（`dom = [selector]`）。

以上两种情况对应的是**用法3**，将dom对象或dom集合转化为 `z` 对象

如果为标签（`fragmentRE.test(selector)`），执行跟**分支1**一模一样的代码。这里判断在上面已经做过了，为什么要再来一次呢？我也不太明白，有明白的可以跟我说下。

经过一轮又一轮的判断和 `selector` 重置，现在终于可以调用 `z` 函数了: `zepto.Z(dom, selector)` ，`init` 的最后，将收集到的 `dom` 集合和对应的 `selector` 传入 `Z` 函数，返回 `Z` 对象。

## zepto.fragment

```javascript
zepto.fragment = function(html, name, properties) {
  var dom, nodes, container
  if (singleTagRE.test(html)) dom = $(document.createElement(RegExp.$1))
  if (!dom) {
    if (html.replace) html = html.replace(tagExpanderRE, "<$1></$2>")
    if (name === undefined) name = fragmentRE.test(html) && RegExp.$1
    if (!(name in containers)) name = '*'
    container = containers[name]
    container.innerHTML = '' + html
    dom = $.each(slice.call(container.childNodes), function() {
      container.removeChild(this)
    })
  }
  if (isPlainObject(properties)) {
    nodes = $(dom)
    $.each(properties, function(key, value) {
      if (methodAttributes.indexOf(key) > -1) nodes[key](value)
      else nodes.attr(key, value)
        })
  }
  return dom
}
```

`fragment` 的作用的是将html片断转换成dom数组形式。

首先判断是否为标签的形式 `singleTagRE.test(html)` （如`<div></div>`）, 如果是，则采用该标签名来创建dom对象 `dom = $(document.createElement(RegExp.$1))`，不用再作其他处理。`singleTagRE = /^<(\w+)\s*\/?>(?:<\/\1>|)$/`。

如果尚未获取到 `dom`，接着进行：

```javascript
if (html.replace) html = html.replace(tagExpanderRE, "<$1></$2>")
```

这段是对 `html` 进行修复，如`<p class="test" />` 修复成 `<p class="test" /></p>` 。正则表达式为 `tagExpanderRE = /<(?!area|br|col|embed|hr|img|input|link|meta|param)(([\w:]+)[^>]*)\/>/ig` 

```javascript
if (name === undefined) name = fragmentRE.test(html) && RegExp.$1
```

如果没有指定标签名，则获取标签名。如传入 `<div>test</div>` ，获取到的 `name` 为 `div`

```javascript
if (!(name in containers)) name = '*'
    container = containers[name]
    container.innerHTML = '' + html
    dom = $.each(slice.call(container.childNodes), function() {
      container.removeChild(this)
    })
  }
// containers 已经开头定义，如下
table = document.createElement('table'),
        tableRow = document.createElement('tr'),
containers = {
  'tr': document.createElement('tbody'),
  'tbody': table,
  'thead': table,
  'tfoot': table,
  'td': tableRow,
  'th': tableRow,
  '*': document.createElement('div')
}
```

检测 `name` 是否为特殊的元素，如 `tr` 要用 `tbody` 包裹，其他的元素用 `div` 包裹。包裹元素的 `childNodes` 即为所需要获取的 `dom` 。

```javascript
if (isPlainObject(properties)) {
    nodes = $(dom)
    $.each(properties, function(key, value) {
      if (methodAttributes.indexOf(key) > -1) nodes[key](value)
      else nodes.attr(key, value)
        })
  }
// methodAttributes 在上面已经定义，定义如下
methodAttributes = ['val', 'css', 'html', 'text', 'data', 'width', 'height', 'offset']
```

如果属性值为纯对象，则给元素设置属性。

如果所需设置的属性，zepto已经定义了相应的方法，则调用zepto对应的方法，否则统一调用zepto的`attr` 方法设置属性。

最后将 `dom` 返回

## 系列文章

1. [读Zepto源码之代码结构](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E4%BB%A3%E7%A0%81%E7%BB%93%E6%9E%84.md)
2. [读 Zepto 源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md)
3. [读Zepto源码之工具函数](https://github.com/yeyuqiudeng/reading-zepto/blob/a4d6ad99c57047beae2b652b4d2cbb380599a524/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md)

## 参考

* [各浏览器对document.getElementById等方法的实现差异解析](http://www.jb51.net/article/44147.htm)
* [Node.nodeType](https://developer.mozilla.org/en-US/docs/Web/API/Node/nodeType)

## License

[署名-非商业性使用-禁止演绎 4.0 国际 (CC BY-NC-ND 4.0)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://segmentfault.com/img/bVCJ55?w=430&h=430) 

作者：对角另一面