# 读Zepto源码之内部方法

## 数组方法

### 定义

```javascript
var emptyArray = []
	concat = emptyArray.concat
    filter = emptyArray.filter
    slice = emptyArray.slice
```

zepto 一开始就定义了一个空数组 `emptyArray`，定义这个空数组是为了取得数组的 `concat`、`filter`、`slice` 方法

### compact

```javascript
function compact(array) {
  return filter.call(array, function(item) {
    return item != null
  })
}
```

删除数组中的 `null` 和 `undefined` 

这里用的是数组的 `filter` 方法，过滤出 `item != null` 的元素，组成新的数组。这里删除掉 `null` 很容易理解，为什么还可以删除 `undefined` 呢？这是因为这里用了 `!=` ，而不是用 `!==` ，用 `!=` 时， `null` 各 `undefined` 都会先转换成 `false` 再进行比较。

关于 `null` 和 `undefined` 推荐看看这篇文章： [undefined与null的区别](http://www.ruanyifeng.com/blog/2014/03/undefined-vs-null.html)

### flatten

```javascript
function flatten(array) {
  return array.length > 0 ? $.fn.concat.apply([], array) : array
}
```

将数组扁平化，例如将数组 `[1,[2,3],[4,5],6,[7,[89]]` 变成 `[1,2,3,4,5,6,7,[8,9]]` ,这个方法只能展开一层，多层嵌套也只能展开一层。

这里，我们先把 `$.fn.concat` 等价于数组的原生方法 `concat`，后面的章节也会分析  `$.fn.concat` 的。

这里比较巧妙的是利用了 `apply` ，`apply` 会将 `array` 中的 `item` 当成参数，`concat.apply([], [1,2,3,[4,5]])` 相当于 `[].concat(1,2,3,[4,5])`，这样数组就扁平化了。

### uniq

```javascript
uniq = function(array) {
  return filter.call(array, function(item, idx) {
    return array.indexOf(item) == idx
  })
}
```

数组去重。

数组去重的原理是检测 `item` 在数组中第一次出现的位置是否和 `item` 所处的位置相等，如果不相等，则证明不是第一次出现，将其过滤掉。

## 字符串方法

### camelize

```javascript
camelize = function(str) {
  return str.replace(/-+(.)?/g, function(match, chr) {
    return chr ? chr.toUpperCase() : ''
  })
}
```

将 `word-word` 的形式的字符串转换成 `wordWord` 的形式， `-` 可以为一个或多个。

正则表达式匹配了一个或多个 `-` ，捕获组是捕获 `-` 号后的第一个字母，并将字母变成大写。

### dasherize

```javascript
function dasherize(str) {
    return str.replace(/::/g, '/')
           .replace(/([A-Z]+)([A-Z][a-z])/g, '$1_$2')
           .replace(/([a-z\d])([A-Z])/g, '$1_$2')
           .replace(/_/g, '-')
           .toLowerCase()
  }
```

将驼峰式的写法转换成连字符 `-` 的写法。

例如 `a = A6DExample::Before`

第一个正则表达式是将字符串中的 `::` 替换成 `/` 。`a` 变成 A6DExample/Before

第二个正则是在出现一次或多次大写字母和出现一次大写字母和连续一次或多次小写字母之间加入 `_`。`a` 变成 A6D_Example/Before

第三个正则是将出现一次小写字母或数字和出现一次大写字母之间加上 `_`。`a` 变成`A6_D_Example/Before`

第四个正则表达式是将 `_` 替换成 `-`。`a` 变成`A6-D-Example/Before`

最后是将所有的大写字母转换成小写字母。`a` 变成 `a6-d-example/before`

我对正则不太熟悉，正则解释部分参考自:[zepto源码--compact、flatten、camelize、dasherize、uniq--学习笔记](http://www.cnblogs.com/zhuhuoxingguang/p/6006743.html)

## 数据类型检测

### 定义

```javascript
class2type = {},
toString = class2type.toString,

  // Populate the class2type map
$.each("Boolean Number String Function Array Date RegExp Object Error".split(" "), function(i, name) {
  class2type["[object " + name + "]"] = name.toLowerCase()
})
```

$.each 函数后面的文章会讲到，这段代码是将基本类型挂到 `class2type` 对象上。`class2type` 将会是如下的形式：

```javascript
class2type = {
  "[object Boolean]": "boolean",
  "[object Number]": "number"
  ...
} 
```

### type

```javascript

function type(obj) {
  return obj == null ? String(obj) :
  class2type[toString.call(obj)] || "object"
}
```

`type` 函数返回的是数据的类型。

如果 `obj == null` ，也就是 `null` 和 `undefined`，返回的是字符串 `null` 或 `undefined`

否则调用 `Object.prototype.toString` （`toString = class2type.toString`）方法，将返回的结果作为 `class2type` 的 key 取值。`Object.prototype.toString` 对不同的数据类型会返回形如 `[object Boolean]` 的结果。

如果都不是以上情况，默认返回 `object` 类型。

### isFunction & isObject

```javascript
function isFunction(value) {
  return type(value) === 'function'
}
function isObject(obj) {
  return type(obj) == 'object'
}
```

调用 `type` 函数，判断返回的类型字符串，就知道是什么数据类型了

### isWindow

```javascript
function isWindow(obj) {
  return obj != null && obj == obj.window
}
```

判断是否为浏览器的 `window` 对象

要为 `window` 对象首先要满足的条件是不能为 `null` 或者 `undefined`， 并且 `obj.window` 为自身的引用。

### isDocument

```javascript
function isDocument(obj) {
  return obj != null && obj.nodeType == obj.DOCUMENT_NODE
}
```

判断是否为 `document` 对象

节点上有 `nodeType` 属性，每个属性值都有对应的常量。`document` 的 `nodeType` 值为 `9` ，常量为 `DOCUMENT_NODE`。

具体见：[MDN文档：Node.nodeType](https://developer.mozilla.org/en-US/docs/Web/API/Node/nodeType)

### isPlainObject

```javascript
function isPlainObject(obj) {
  return isObject(obj) && !isWindow(obj) && Object.getPrototypeof(obj) == Object.prototype
}
```

判断是否为纯粹的对象

纯粹对象首先必须是对象 `isObject(obj)`

并且不是 `window` 对象 `!isWindow(obj)`

并且原型要和 `Object` 的原型相等

### isArray

```javascript
isArray = Array.isArray || 
  		 function(object) { return object instanceof Array}
```

这个方法来用判断是否为数组类型。

如果浏览器支持数组的 `isArray` 原生方法，就采用原生方法，否则检测数据是否为 `Array` 的实例。

我们都知道，`instanceof` 的检测的原理是查找实例的 `prototype` 是否在构造函数的原型链上，如果在，则返回 `true`。 所以用 `instanceof` 可能会得到不太准确的结果。例如：

index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<script>
		window.onload = function () {
			var fwindow = window.framePage.contentWindow // frame 页面的window对象
			var fArray = fwindow.Array  // frame 页面的Array
			var fdata = fwindow.data  // frame 页面的 data [1,2,3]
			console.log(fdata instanceof fArray) // true
			console.log(fdata instanceof Array) // false
		}
	</script>
	<title>Document</title>
</head>
<body>
	<iframe id="framePage" src="frame.html" frameborder="0"></iframe>
</body>
</html>
```

frame.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<title>Document</title>
	<script>
		window.data = [1,2,3]
	</script>
</head>
<body>
	<p>frame page</p>
</body>
</html>
```

由于 `iframe` 是在独立的环境中运行的，所以 `fdata instanceof Array ` 返回的 `false` 。

在 MDN 上看到，可以用这样的 ployfill 来使用 isArray

```javascript
if (!Array.isArray) {
  Array.isArray = function(arg) {
    return Object.prototype.toString.call(arg) === '[object Array]'
  }
}
```

也就是说，`isArray` 可以修改成这样：

```javascript
isArray = Array.isArray || 
  		 function(object) { return Object.prototype.toString.call(object) === '[object Array]'}
```

为什么 zepto 不这样写呢？知道的可以留言告知下。

### likeArray

```javascript
function likeArray(obj) {
  var length = !!obj &&   // obj必须存在
      			'length' in obj && // obj 中必须存在 length 属性
      			obj.length, // 返回 length的值
      type = $.type(obj) // 调用 type 函数，返回 obj 的数据类型。这里我有点不太明白，为什么要覆盖掉上面定义的 type 函数呢？再定义多一个变量，直接调用 type 函数不好吗？

  return 'function' != type &&  // 不为function类型
    	!isWindow(obj) &&  // 并且不为window类型
    	(
    		'array' == type || length === 0 || // 如果为 array 类型或者length 的值为 0，返回true
    (typeof length == 'number' && length > 0 && (length - 1) in obj)  // 或者 length 为数字，并且 length的值大于零，并且 length - 1 为 obj 的 key
  )
}
```

判断是否为数据是否为类数组。

类数组的形式如下：

```javascript
likeArrayData = {
  '0': 0,
  '1': 1,
  "2": 2
  length: 3
}
```

可以看到，类数组都有 `length` 属性，并且 `key` 为按`0,1,2,3` 顺序的数字。

代码已经有注释了，这里再简单总结下

首先将 `function`类型和 `window` 对象排除

再将 type 为 `array` 和 `length === 0` 的认为是类数组。type 为 `array` 比较容易理解，`length === 0` 其实就是将其看作为空数组。

最后一种情况必须要满足三个条件：

1. `length` 必须为数字
2. `length` 必须大于 `0` ，表示有元素存在于类数组中
3. key `length - 1` 必须存在于 `obj` 中。我们都知道，数组最后的 `index` 值为 `length -1` ，这里也是检查最后一个 `key` 是否存在。

## 系列文章

1. [读Zepto源码之代码结构](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E4%BB%A3%E7%A0%81%E7%BB%93%E6%9E%84.md)

## 参考

* [MDN文档：Array.isArray()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/isArray)
* [MDN文档：Function.prototype.apply() ](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply)
* [MDN文档：Node.nodeType](https://developer.mozilla.org/en-US/docs/Web/API/Node/nodeType)
* [undefined与null的区别](http://www.ruanyifeng.com/blog/2014/03/undefined-vs-null.html)
* [zepto源码--compact、flatten、camelize、dasherize、uniq--学习笔记](http://www.cnblogs.com/zhuhuoxingguang/p/6006743.html)



最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：

  ![](https://segmentfault.com/img/bVCJ55?w=430&h=430)