# 读Zepto源码之IOS3模块

`IOS3` 模块是针对 `IOS` 的兼容模块，实现了两个常用方法的兼容，这两个方法分别是 `trim` 和 `reduce` 。 

读 Zepto 源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## trim

```javascript
if (String.prototype.trim === undefined) // fix for iOS 3.2
  String.prototype.trim = function(){ return this.replace(/^\s+|\s+$/g, '') }
```

看注释， `trim` 是为了兼容 `ios3.2` 的。

也是常规的做法，如果 `String` 的 `prototype` 上没有 `trim` 方法，则自己实现一个。

实现的方式也简单，就是用正则将开头和结尾的空格去掉。`^\s+` 这段是匹配开头的空格，`\s+$` 是匹配结尾的空格。

## reduce

```javascript
// For iOS 3.x
// from https://developer.mozilla.org/en/JavaScript/Reference/Global_Objects/Array/reduce
if (Array.prototype.reduce === undefined)
  Array.prototype.reduce = function(fun){
    if(this === void 0 || this === null) throw new TypeError()
    var t = Object(this), len = t.length >>> 0, k = 0, accumulator
    if(typeof fun != 'function') throw new TypeError()
    if(len == 0 && arguments.length == 1) throw new TypeError()

    if(arguments.length >= 2)
      accumulator = arguments[1]
    else
      do{
        if(k in t){
          accumulator = t[k++]
          break
        }
        if(++k >= len) throw new TypeError()
      } while (true)

    while (k < len){
      if(k in t) accumulator = fun.call(undefined, accumulator, t[k], k, t)
      k++
    }
    return accumulator
  }
```

### 用法与参数

要理解这段代码，先来看一下 `reduce` 的用法和参数：

**用法**： 

> arr.reduce(callback[, initialValue])

**参数**：

* callback: 回调函数，有如下参数
  * accumulator: 上一个回调函数返回的值或者是初始值（`initialValue`）
  * currentValue: 当前值
  * currentIndex: 当前值在数组中的索引
  * array: 调用 `reduce` 的数组
* initialValue: 初始值，如果没有提供，则为数组的第一项。如果数组为空数组，而又没有提供初始值时，会报错

### 检测参数

```javascript
if(this === void 0 || this === null) throw new TypeError()
var t = Object(this), len = t.length >>> 0, k = 0, accumulator
if(typeof fun != 'function') throw new TypeError()
if(len == 0 && arguments.length == 1) throw new TypeError()
```

首先检测是否为 `undefined` 或者 `null` ，如果是，则报类型错误。这里有一点值得注意的，判断是否为 `undefined` 时，用了 `void 0` 的返回值，因为 `void` 操作符返回的结果都为 `undefined` ，这是为了避免 `undefined` 被重新赋值，出现误判的情况。

接下来，将数组转换成对象，用变量 `t` 来保存，后面会看到，遍历用的是 `for...in` 来处理。为什么不直接用 `for` 来处理数组呢？因为 `reduce` 不会处理稀疏数组，所以转换要转换成对象来处理。

数组长度用 `len` 来保存，这里使用了无符号位右移操作符 `>>>` ，确保 `len` 为非负整数。

用 `k` 来保存当前索引，`accumulator` 为返回值。

接下来，检测回调函数 `fun` 是否为 `function` ，如果不是，抛出类型错误。

 在数组为空，并且又没有提供初始值（即只有一个参数 `fun`）时，抛出类型错误。

### accumulator初始值

```javascript
if(arguments.length >= 2)
  accumulator = arguments[1]
else
  do{
    if(k in t){
      accumulator = t[k++]
      break
    }
    if(++k >= len) throw new TypeError()
  } while (true)
```

如果参数至少有两项，则 `accumulator` 的初始值很简单，就是 `arguments[1]` ，即 `initialValue`。

如果没有提供初始值，则叠加索引，直到找到在对象 `t` 中存在的索引。注意这里用了 `do...while`，所以最终结果，要么是报类型错误，要么 `accumulator` 能获取到值。

这段还巧妙地用了 `++k` 和 `k++` 。如果 `k` 在对象 `t` 中存在时，则赋值给 `accumulator` 后 `k` 再自增，否则用 `k` 自增后再和 `len` 比较，如果超出 `len` 的长度，则报错，因为不存在下一个可以赋给 `accumulator` 的值。



## 系列文章

《[reading-zepto](https://yeyuqiudeng.gitbooks.io/reading-zepto/content/)》

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
15. [读Zepto源码之Assets模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8Bassets%E6%A8%A1%E5%9D%97.md)
16. [读Zepto源码之Selector模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BSelector%E6%A8%A1%E5%9D%97.md)
17. [读Zepto源码之Touch模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BTouch%E6%A8%A1%E5%9D%97.md)
18. [读Zepto源码之Gesture模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BGesture%E6%A8%A1%E5%9D%97.md)


### 附文

* [译：怎样处理 Safari 移动端对图片资源的限制](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E9%99%84%EF%BC%9A%E6%80%8E%E6%A0%B7%E5%A4%84%E7%90%86%20Safari%20%E7%A7%BB%E5%8A%A8%E7%AB%AF%E5%AF%B9%E5%9B%BE%E7%89%87%E8%B5%84%E6%BA%90%E7%9A%84%E9%99%90%E5%88%B6.md)


## 参考

* [Array.prototype.reduce()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce)

## License

[署名-非商业性使用-禁止演绎 4.0 国际 (CC BY-NC-ND 4.0)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://user-gold-cdn.xitu.io/2017/5/30/76626b0be42083d36b36f4a117dc1873) 

作者：对角另一面