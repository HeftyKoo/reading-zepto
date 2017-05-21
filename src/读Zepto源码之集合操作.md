# 读Zepto源码之集合操作

接下来几个篇单，都会解读 zepto 中的跟 `dom` 相关的方法，也即源码 `$.fn` 对象中的方法。

读Zepto源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## .forEach()

```javascript
forEach: emptyArray.forEach
```

因为 zepto 的 `dom` 集合是类数组，所以这里只是简单地复制了数组的 `forEach` 方法。

具体的 `forEach` 的用法见文档:[Array.prototype.forEach()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach?v=example)

##  .reduce()

```javascript
reduce: emptyArray.reduce
```

简单地复制了数组的 `reduce` 方法。

具体的 `reduce` 的用法见文档:[Array.prototype.reduce()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce?v=example)

## .push()

```javascript
push: emptyArray.push
```

简单地复制了数组的 `push` 方法。

具体的 `push` 的用法见文档:[Array.prototype.push()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/push?v=example)

## .sort()

```javascript
sort: emptyArray.sort
```

简单地复制了数组的 `sort` 方法。

具体的 `sort` 的用法见文档:[Array.prototype.sort()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort?v=example)

## .splice()

```javascript
splice: emptyArray.splice
```

简单地复制了数组的 `splice` 方法。

具体的 `splice` 的用法见文档:[Array.prototype.splice()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/splice?v=example)

## .indexOf()

```javascript
indexOf: emptyArray.indexOf
```

简单地复制了数组的 `indexOf` 方法。

具体的 `indexOf` 的用法见文档:[Array.prototype.indexOf()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/indexOf)

## .get()

```javascript
get: function(idx) {
  return idx === undefined ? slice.call(this) : this[idx >= 0 ? idx : idx + this.length]
},
```

这个方法用来获取指定索引值的元素。

不传参（`idx === undefined`）时，不传参调用数组的 `slice` 方法，将集合中的所有元素返回。

当传递的参数大于或等于零（`idx`）时，返回相应索引值的元素 `this[idx]` ，如果为负数，则倒数返回`this.[idx + this.length]`。

例如  `$('li').get(-1)` 返回的是倒数第1个元素，也即最后一个元素

## .toArray()

```javascript
toArray: function() { return this.get() }
```

`toArray` 方法是将元素的类数组变成纯数组。`toArray` 内部不传参调用 `get` 方法，上面已经分析了，当不传参数时，`get` 方法调用的是数组方法 `slice`， 返回的自然就是纯数组了。

## .size()

```javascript
size: function() {
  return this.length
}
```

`size` 方法返回的是集合中的 `length` 属性，也即集合中元素的个数。

## .concat()

```javascript
concat: function() {
  var i, value, args = []
  for (i = 0; i < arguments.length; i++) {
    value = arguments[i]
    args[i] = zepto.isZ(value) ? value.toArray() : value
  }
  return concat.apply(zepto.isZ(this) ? this.toArray() : this, args)
},
```

数组中也有对应的 `concat` 方法，为什么不能像上面的方法那样直接调用呢？

这是因为 `$.fn` 其实是一个类数组对象，并不是真正的数组，如果直接调用 `concat` 会直接把整个 `$.fn` 当成数组的一个 `item` 合并到数组中。

```javascript
for (i = 0; i < arguments.length; i++) {
  value = arguments[i]
  args[i] = zepto.isZ(value) ? value.toArray() : value
}
```

这段是对每个参数进行判断，如果参数是 `zepto` 的集合（`zepto.isZ(value)`），就先调用 `toArray` 方法，转换成纯数组。

```javascript
return concat.apply(zepto.isZ(this) ? this.toArray() : this, args)
```

这段同样对 `this` 进行了判断，如果为 `zepto` 集合，也先转换成数组。所以调用 `concat` 后返回的是纯数组，不再是 `zepto` 集合。

## .map()

```javascript
map: function(fn) {
  return $($.map(this, function(el, i) { return fn.call(el, i, el) }))
}
```

`map` 方法的内部调用的是 `zepto` 的工具函数 `$.map` ，这在之前已经在《[读Zepto源码之工具函数](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md#map)》做过了分析。

```javascript
return fn.call(el, i, el)
```

`map` 方法对回调也做了包装，`call` 的第一个参数为 `el` ，因此可以在 `map` 的回调中通过 `this` 来拿到每个元素。

`map` 方法对 `$.map` 返回的数组调用了 `$()` 方法，将返回的数组再次包装成 `zepto` 对象，因此调用 `map` 方法后得到的数组，同样具有 `zepto` 集合中的方法。

## .slice() 

```javascript
slice: function() {
  return $(slice.apply(this, arguments))
}
```

`slice` 同样没有直接用数组的原生方法，也像 `map` 方法一样，将返回的数组再次包装成 `zepto` 对象。

## .each()

```javascript
each: function(callback) {
  emptyArray.every.call(this, function(el, idx) {
    return callback.call(el, idx, el) !== false
  })
  return this
},
```

`zepto` 的 `each` 方法比较巧妙，在方法内部，调用的其实是数组的 `every` 方法，`every` 遇到 `false` 时就会中止遍历，`zepto` 也正是利用 `every` 这种特性，让 `each` 方法也具有了中止遍历的能力，当 `callback` 返回的值为布尔值 `false` 时，中止遍历，注意这里用了 `!==`，因为 `callback` 如果没有返回值时，得到的值会是 `undefined` ，这种情况是需要排除的。

同样，`each` 的回调中也是可以用 `this` 拿到每个元素的。

注意，`each` 方法最后返回的是 `this`， 所以在 `each` 调用完后，还可以继续调用 集合中的其他方法，这就是 `zepto` 的链式调用，这个跟 `map` 方法中返回 `zepto` 集合的原理差不多，只不过 `each` 返回的是跟原来一样的集合，`map` 方法返回的是映射后的集合。

## .add()

```javascript
add: function(selector, context) {
  return $(uniq(this.concat($(selector, context))))
}
```

`add` 可以传递两个参数，`selector` 和 `context` ，即选择器和上下文。

`add` 调用 `$(selector, context)` 来获取符合条件的集合元素，这在上篇文章《[读Zepto源码之神奇的$](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E7%A5%9E%E5%A5%87%E7%9A%84%24.md#的实现-zeptoinit-函数)》已经有详细的论述。

然后调用 `concat` 方法来合并两个集合，用内部方法 `uniq` 来过滤掉重复的项，`uniq` 方法在《[读Zepto源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md#uniq)》已经有论述。最后也是返回一个 `zepto` 集合。 

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## 系列文章

1. [读Zepto源码之代码结构](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E4%BB%A3%E7%A0%81%E7%BB%93%E6%9E%84.md)
2. [读 Zepto 源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md)
3. [读Zepto源码之工具函数](https://github.com/yeyuqiudeng/reading-zepto/blob/a4d6ad99c57047beae2b652b4d2cbb380599a524/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md)
4. [读Zepto源码之神奇的$](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E7%A5%9E%E5%A5%87%E7%9A%84%24.md)

## License

[![License: CC BY-NC-ND 4.0](https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://segmentfault.com/img/bVCJ55?w=430&h=430) 

作者：对角另一面