# 读Zepto源码之Selector模块

`Selector` 模块是对 `Zepto` 选择器的扩展，使得 `Zepto` 选择器也可以支持部分 `CSS3` 选择器和 `eq` 等 `Zepto` 定义的选择器。

在阅读本篇文章之前，最好先阅读《[读Zepto源码之神奇的$](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E7%A5%9E%E5%A5%87%E7%9A%84%24.md)》。

读 Zepto 源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## 辅助方法

### visible

```javascript
function visible(elem){
  elem = $(elem)
  return !!(elem.width() || elem.height()) && elem.css("display") !== "none"
}
```

判断元素是否可见。

可见的标准是元素有宽或者高，并且 `display` 值不为 `none`。

### filters

```javascript
var filters = $.expr[':'] = {
  visible:  function(){ if (visible(this)) return this },
  hidden:   function(){ if (!visible(this)) return this },
  selected: function(){ if (this.selected) return this },
  checked:  function(){ if (this.checked) return this },
  parent:   function(){ return this.parentNode },
  first:    function(idx){ if (idx === 0) return this },
  last:     function(idx, nodes){ if (idx === nodes.length - 1) return this },
  eq:       function(idx, _, value){ if (idx === value) return this },
  contains: function(idx, _, text){ if ($(this).text().indexOf(text) > -1) return this },
  has:      function(idx, _, sel){ if (zepto.qsa(this, sel).length) return this }
}
```

定义了一系列的过滤函数，返回符合条件的元素。这些过滤函数会将集合中符合条件的元素过滤出来，是实现相关选择器的核心。

* visible: 过滤可见元素，匹配 `el:visible` 选择器
* hidden: 过滤不可见元素， 匹配 `el:hidden` 选择器
* selected: 过滤选中的元素，匹配 `el:selected` 选择器
* checked: 过滤勾选中的元素，匹配 `el:checked` 选择器
* parent: 返回至少包含一个子元素的元素，匹配 `el:parent` 选择器
* first: 返回第一个元素，匹配 `el:first` 选择器
* last: 返回最后一个元素，匹配 `el:last` 选择器
* eq: 返回指定索引的元素，匹配 `el:eq(index)` 选择器
* contains: 返回包含指定文本的元素，匹配 `el:contains(text)`
* has: 返回匹配指定选择器的元素，匹配 `el:has(sel)`

### process

```javascript
var filterRe = new RegExp('(.*):(\\w+)(?:\\(([^)]+)\\))?$\\s*'),
function process(sel, fn) {
  sel = sel.replace(/=#\]/g, '="#"]')
  var filter, arg, match = filterRe.exec(sel)
  if (match && match[2] in filters) {
    filter = filters[match[2]], arg = match[3]
    sel = match[1]
    if (arg) {
      var num = Number(arg)
      if (isNaN(num)) arg = arg.replace(/^["']|["']$/g, '')
      else arg = num
    }
  }
  return fn(sel, filter, arg)
}
```

`process` 方法是根据参数 `sel`，分解出选择器、伪类名和伪类参数（如 `eq` 、 `has` 的参数），根据伪类来选择对应的 `filter` ，传递给回调函数 `fn` 。

分解参数最主要靠的是 `filterRe` 这条正则，正则太过复杂，我也不太懂，很难解释，不过用正则可视化网站 [regexper.com](https://regexper.com)，可以很清晰地看到，正则分成三大组，第一组匹配的是 `:` 前面的选择器，第二组匹配的是伪类名，第三组匹配的是伪类参数。

```javascript
sel = sel.replace(/=#\]/g, '="#"]')
```

这段是处理 `a[href^=#]` 的情况，其实就是将 `#` 包在 `""` 里面，以符合标准的属性选择器，这是 `Zepto` 的容错能力。 这个选择器不会匹配到 `filters` 上的过滤函数，最后调用的是 `querySelectorAll` 方法，具体见《[读Zepto源码之神奇的$](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E7%A5%9E%E5%A5%87%E7%9A%84%24.md)》对 `qsa` 函数的分析。

```javascript
if (match && match[2] in filters) {
  filter = filters[match[2]], arg = match[3]
  sel = match[1]
  ...
}
```

`match[2]` 也即第二组匹配的是伪类名，也是对应 `filters` 中的 `key` 值，伪类名存在于 `filters` 中时，则将选择器，伪类名和伪类参数存入对应的变量。

```javascript
if (arg) {
  var num = Number(arg)
  if (isNaN(num)) arg = arg.replace(/^["']|["']$/g, '')
  else arg = num
}
```

如果伪类的参数不可以用 `Number` 转换，则参数为字符串，用正则将字符串前后的 `"` 或 `'` 去掉，再赋值给 `arg`.

```javascript
return fn(sel, filter, arg)
```

最后执行回调，将解释出来的参数传入回调函数中，将执行结果返回。

## 重写的方法

###qsa

```javascript
var zepto = $.zepto, oldQsa = zepto.qsa, oldMatches = zepto.matches，
	childRe  = /^\s*>/,
    classTag = 'Zepto' + (+new Date())

zepto.qsa = function(node, selector) {
  return process(selector, function(sel, filter, arg){
    try {
      var taggedParent
      if (!sel && filter) sel = '*'
      else if (childRe.test(sel))
        taggedParent = $(node).addClass(classTag), sel = '.'+classTag+' '+sel

      var nodes = oldQsa(node, sel)
      } catch(e) {
        console.error('error performing selector: %o', selector)
        throw e
      } finally {
        if (taggedParent) taggedParent.removeClass(classTag)
      }
    return !filter ? nodes :
    zepto.uniq($.map(nodes, function(n, i){ return filter.call(n, i, nodes, arg) }))
  })
}
```

改过的 `qsa` 调用的是 `process` 方法，在回调函数中处理大部分逻辑。

思路是通过选择器获取到所有节点，然后再调用对应伪类的对应方法来过滤出符合条件的节点。

#### 处理选择器，根据选择器获取节点

```javascript
var taggedParent
if (!sel && filter) sel = '*'
else if (childRe.test(sel))
  taggedParent = $(node).addClass(classTag), sel = '.'+classTag+' '+sel

var nodes = oldQsa(node, sel)
```

如果选择器和过滤器都不存在，则将 `sel` 设置 `*` ，即获取所有元素。

如果 `>sel` 形式的选择器，查找所有子元素。

这里的做法是，向元素 `node` 中添加唯一的样式名 `classTag`，然后用唯一样式名和选择器拼接成子元素选择器。

最后调用原有的 `qsa` 函数 `oldQsa`  来获取符合选择器的所有元素。

#### 清理所添加的样式

```javascript
if (taggedParent) taggedParent.removeClass(classTag)
```

如果存在 `taggedParent` ，则将元素上的 `classTag` 清理掉。

#### 调用对应的过滤器，过滤元素

```javascript
return !filter ? nodes :
	zepto.uniq($.map(nodes, function(n, i){ return filter.call(n, i, nodes, arg) }))
```

如果没有过滤器，则将所有元素返回，如果存在过滤器，则遍历集合，调用对应的过滤器获取元素，并将新集合的元素去重。

### matches

```javascript
zepto.matches = function(node, selector){
  return process(selector, function(sel, filter, arg){
    return (!sel || oldMatches(node, sel)) &&
      (!filter || filter.call(node, null, arg) === node)
  })
}
```

`matches` 也是调用 `process` 方法，这里很巧妙地用了 `||` 和 `&&` 的短路操作。

其实要做的事情就是，如果可以用 `oldMatches` 匹配，则使用 `oldMatches` 匹配的结果，否则使用过滤器过滤出来的结果。

## 系列文章

1. [读Zepto源码之代码结构](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E4%BB%A3%E7%A0%81%E7%BB%93%E6%9E%84.md)
2. [读Zepto源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md)
3. [读Zepto源码之工具函数](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md)
4. [读Zepto源码之神奇的$](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E7%A5%9E%E5%A5%87%E7%9A%84%24.md)
5. [读Zepto源码之集合操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E6%93%8D%E4%BD%9C.md)
6. [读Zepto源码之集合元素查找](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E9%9B%86%E5%90%88%E5%85%83%E7%B4%A0%E6%9F%A5%E6%89%BE.md)
7. [读Zepto源码之操作DOM](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E6%93%8D%E4%BD%9CDOM.md)
8. [读Zepto源码之样式操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E6%A0%B7%E5%BC%8F%E6%93%8D%E4%BD%9C.md)
9. [读Zepto源码之属性操作](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B1%9E%E6%80%A7%E6%93%8D%E4%BD%9C.md)
10. [读Zepto源码之Event模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BEvent%E6%A8%A1%E5%9D%97.md)
11. [读Zepto源码之IE模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BIE%E6%A8%A1%E5%9D%97.md)
12. [读Zepto源码之Callbacks模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BCallbacks%E6%A8%A1%E5%9D%97.md)
13. [读Zepto源码之Deferred模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BDeferred%E6%A8%A1%E5%9D%97.md)
14. [读Zepto源码之Ajax模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BAjax%E6%A8%A1%E5%9D%97.md)
15. [读Zepto源码之assets模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8Bassets%E6%A8%A1%E5%9D%97.md)



## 参考

* [try-catch语句的“伪块作用域”](https://www.web-tinker.com/article/20331.html)

## License

[![License: CC BY-NC-ND 4.0](https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://user-gold-cdn.xitu.io/2017/5/30/76626b0be42083d36b36f4a117dc1873) 

作者：对角另一面