# 读zepto源码之工具函数

Zepto 提供了丰富的工具函数，下面来一一解读。

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## $.extend

`$.extend` 方法可以用来扩展目标对象的属性。目标对象的同名属性会被源对象的属性覆盖。

`$.extend` 其实调用的是内部方法 `extend`， 所以我们先看看内部方法 `extend` 的具体实现。

```javascript
function extend(target, source, deep) {
        for (key in source)  // 遍历源对象的属性值
            if (deep && (isPlainObject(source[key]) || isArray(source[key]))) { // 如果为深度复制，并且源对象的属性值为纯粹对象或者数组
                if (isPlainObject(source[key]) && !isPlainObject(target[key])) // 如果为纯粹对象
                    target[key] = {}  // 如果源对象的属性值为纯粹对象，并且目标对象对应的属性值不为纯粹对象，则将目标对象对应的属性值置为空对象
                if (isArray(source[key]) && !isArray(target[key])) // 如果源对象的属性值为数组，并且目标对象对应的属性值不为数组，则将目标对象对应的属性值置为空数组
                    target[key] = []
                extend(target[key], source[key], deep) // 递归调用extend函数
            } else if (source[key] !== undefined) target[key] = source[key]  // 不对undefined值进行复制
    }
```

`extend` 的第一个参数 `taget` 为目标对象， `source` 为源对象， `deep` 表示是否为深度复制。当 `deep` 为 `true` 时为深度复制， `false` 时为浅复制。

1. `extend` 函数用 `for···in` 对 `source` 的属性进行遍历

2. 如果 `deep` 为 `false` 时，只进行浅复制，将 `source` 中不为 `undefined` 的值赋值到 `target` 对应的属性中（注意，这里用的是 `!==`，不是 `!=` ，所以只排除严格为 `undefined` 的值，不包含 `null` ）。如果 `source` 对应的属性值为对象或者数组，会保持该对象或数组的引用。

3. 如果 `deep` 为 `true` ，并且 `source` 的属性值为纯粹对象或者数组时

   3.1. 如果 `source` 的属性为纯粹对象，并且 `target` 对应的属性不为纯粹对象时，将 `target` 的对应属性设置为空对象

   3.2. 如果 `source` 的属性为数组，并且 `target` 对应属性不为数组时，将 `target` 的对应属性设置为空数组

   3.3. 将 `source` 和 `target` 对应的属性及 `deep` 作为参数，递归调用 `extend` 函数，以实现深度复制。

现在，再看看 `$.extend` 的具体实现

```javascript
$.extend = function(target) {
        var deep, args = slice.call(arguments, 1)
        if (typeof target == 'boolean') {
            deep = target
            target = args.shift()
        }
        args.forEach(function(arg) { extend(target, arg, deep) })
        return target
    }
```

 在说原理之前，先来看看 `$.extend` 的调用方式，调用方式如下：

```javascript
$.extend(target, [source, [source2, ...]])
                  或
$.extend(true, target, [source, ...])
```

在 `$.extend` 中，如果不需要深度复制，第一个参数可以是目标对象 `target`, 后面可以有多个 `source` 源对象。如果需要深度复制，第一个参数为 `deep` ，第二个参数为 `target` ，为目标对象，后面可以有多个 `source` 源对象。

`$.extend` 函数的参数设计得很优雅，不需要深度复制时，可以不用显式地将 `deep` 置为 `false`。这是如何做到的呢？

在 `$.extend` 函数中，定义了一个数组 `args`，用来接受除第一个参数外的所有参数。

然后判断第一个参数 `target` 是否为布尔值，如果为布尔值，表示第一个参数为 `deep` ，那么第二个才为目标对象，因此需要重新为 `target` 赋值为 `args.shift()` 。

最后就比较简单了，循环源对象数组 `args`， 分别调用 `extend` 方法，实现对目标对象的扩展。

## $.each

`$.each` 用来遍历数组或者对象，源码如下：

```javascript
$.each = function(elements, callback) {
        var i, key
        if (likeArray(elements)) {  // 类数组
            for (i = 0; i < elements.length; i++)
                if (callback.call(elements[i], i, elements[i]) === false) return elements
        } else { // 对象
            for (key in elements)
                if (callback.call(elements[key], key, elements[key]) === false) return elements
        }

        return elements
    }
```

先来看看调用方式：`$.each(collection, function(index, item){ ... })`

`$.each` 接收两个参数，第一个参数 `elements` 为需要遍历的数组或者对象，第二个 `callback` 为回调函数。

如果 `elements` 为数组，用 `for` 循环，调用 `callback` ，并且将数组索引 `index`  和元素值 `item` 传给回调函数作为参数；如果为对象，用 `for···in` 遍历属性值，并且将属性 `key` 及属性值传给回调函数作为参数。

注意回调函数调用了 `call` 方法，`call` 的第一个参数为当前元素值或当前属性值，所以回调函数的上下文变成了当前元素值或属性值，也就是说回调函数中的 `this` 指向的是 `item` 。这在dom集合的遍历中相当有用。

在遍历的时候，还对回调函数的返回值进行判断，如果回调函数返回 `false` （`if (callback.call(elements[i], i, elements[i]) === false) `） ，立即中断遍历。

`$.each` 调用结束后，会将遍历的数组或对象（ `elements` ）返回。

## $.map

可以遍历数组（类数组）或对象中的元素，根据回调函数的返回值，将返回值组成一个新的数组，并将该数组扁平化后返回，会将 `null` 及 `undefined` 排除。

```javascript
$.map = function(elements, callback) {
        var value, values = [],
            i, key
        if (likeArray(elements))
            for (i = 0; i < elements.length; i++) {
                value = callback(elements[i], i)
                if (value != null) values.push(value)
            }
        else
            for (key in elements) {
                value = callback(elements[key], key)
                if (value != null) values.push(value)
            }
        return flatten(values)
    }
```

先来看看调用方式： `$.map(collection, function(item, index){ ... }) `

`elements` 为类数组或者对象。`callback` 为回调函数。当为类数组时，用 `for` 循环，当为对象时，用 `for···in` 循环。并且将对应的元素（属性值）及索引（属性名）传递给回调函数，如果回调函数的返回值不为 `null` 或者 `undefined` ，则将返回值存入新数组中，最后将新数组扁平化后返回。

## $.camelCase

该方法是将字符串转换成驼峰式的字符串

```javascript
$.camelCase = camelize
```

`$.camelCase` 调用的是内部方法 `camelize` ,该方法在前一篇文章《[读Zepto源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md#camelize)》中已有阐述，本篇文章就不再展开。

## $.contains

用来检查给定的父节点中是否包含有给定的子节点，源码如下：

```javascript
$.contains = document.documentElement.contains ?
        function(parent, node) {
            return parent !== node && parent.contains(node)
        } :
        function(parent, node) {
            while (node && (node = node.parentNode))
                if (node === parent) return true
            return false
        }
```

先来看看调用：`$.contains(parent, node)`

参数 `parent` 为父子点，`node` 为子节点。

`$.contains` 的主体是一个三元表达式，返回的是一个匿名函数。三元表达式的条件是 `document.documentElement.contains`， 用来检测浏览器是否支持 `contains` 方法，如果支持，则直接调用 `contains` 方法，并且将 `parent` 和 `node` 为同一个元素的情况排除。

否则，返回另一外匿名函数。该函数会一直向上寻找 `node` 元素的父元素，如果能找到跟 `parent` 相等的父元素，则返回 `true`， 否则返回 `false`

## $.grep

该函数其实就是数组的 `filter` 函数

```javascript
  $.grep = function(elements, callback) {
       return filter.call(elements, callback)
   }
```

从源码中也可以看出，`$.grep` 调用的就是数组方法 `filter`

## $.inArray

返回指定元素在数组中的索引值

```javascript
 $.inArray = function(elem, array, i) {
        return emptyArray.indexOf.call(array, elem, i)
    }
```

先来看看调用 `$.inArray(element, array, [fromIndex])`

第一个参数 `element` 为指定的元素，第二个参数为 `array` 为数组， 第三个参数 `fromIndex` 为可选参数，表示从哪个索引值开始向后查找。

`$.inArray` 其实调用的是数组的  `indexOf` 方法，所以传递的参数跟 `indexOf` 方法一致。

## $.isArray

判断是否为数组

```javascript
$.isArray = isArray
```

`$.isArray` 调用的是内部方法 `isArray` ，该方法在前一篇文章《[读Zepto源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md#isarray)》中已有阐述。

## $.isFunction

判读是否为函数

```javascript
$.isFunction = isFunction
```

`$.isFunction` 调用的是内部方法 `isFunction` ，该方法在前一篇文章《[读Zepto源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md#isfunction--isobject)》中已有阐述。

## $.isNumeric

是否为数值

```javascript
$.isNumeric = function(val) {
        var num = Number(val), // 将参数转换为Number类型
            type = typeof val
        return val != null && 
          type != 'boolean' &&
            (type != 'string' || val.length) &&
          !isNaN(num) &&
          isFinite(num) 
          || false
    }
```

判断是否为数值，需要满足以下条件

1. 不为 `null`
2. 不为布尔值
3. 不为NaN(当传进来的参数不为数值或如`'123'`这样形式的字符串时，都会转换成NaN)
4. 为有限数值
5. 当传进来的参数为字符串的形式，如`'123'` 时，会用到下面这个条件来确保字符串为数字的形式，而不是如 `123abc` 这样的形式。`(type != 'string' || val.length) && !isNaN(num)` 。这个条件的包含逻辑如下：如果为字符串类型，并且为字符串的长度大于零，并且转换成数组后的结果不为NaN，则断定为数值。（因为 `Number('')` 的值为 `0`）

## $.isPlainObject

是否为纯粹对象，即以 `{}` 常量或 `new Object()` 创建的对象

```javascript
$.isPlainObject = isPlainObject
```

`$.isPlainObject` 调用的是内部方法` isPlainObject` ，该方法在前一篇文章《[读Zepto源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md#isplainobject)》中已有阐述。

## $.isWindow

是否为浏览器的 `window` 对象

```javascript
$.isWindow = isWindow
```

`$.isWindow` 调用的是内部方法 `isWindow` ，该方法在前一篇文章《[读Zepto源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md#iswindow)》中已有阐述。

## $.noop

空函数

```javascript
$.noop = function() {}
```

这个在需要传递回调函数作为参数，但是又不想在回调函数中做任何事情的时候会非常有用，这时，只需要传递一个空函数即可。

## $.parseJSON

将标准JSON格式的字符串解释成JSON

```javascript
if (window.JSON) $.parseJSON = JSON.parse
```

其实就是调用原生的 `JSON.parse`， 并且在浏览器不支持的情况下，`zepto` 还不提供这个方法。

## $.trim

删除字符串头尾的空格

```javascript
$.trim = function(str) {
  return str == null ? "" : String.prototype.trim.call(str)
}
```

如果参数为 `null` 或者 `undefined` ，则直接返回空字符串，否则调用字符串原生的 `trim` 方法去除头尾的空格。

## $.type

类型检测

```javascript
$.type = type
```

`$.type` 调用的是内部方法 `type` ，该方法在前一篇文章《[读Zepto源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md#数据类型检测)》中已有阐述。

能检测的类型有 `"Boolean Number String Function Array Date RegExp Object Error"`

## 系列文章

1. [读Zepto源码之代码结构](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E4%BB%A3%E7%A0%81%E7%BB%93%E6%9E%84.md)
2. [读 Zepto 源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md)

## 参考

* [Zepto中文文档](http://www.css88.com/doc/zeptojs_api/)
* [Node.contains()](https://developer.mozilla.org/en-US/docs/Web/API/Node/contains)
* [Array.prototype.indexOf()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/indexOf?v=example)
* [String.prototype.trim()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/Trim)

## License

[![License: CC BY-NC-ND 4.0](https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://segmentfault.com/img/bVCJ55?w=430&h=430) 

作者：对角另一面