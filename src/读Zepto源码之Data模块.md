# 读Zepto源码之Data模块

`Zepto` 的 `Data` 模块用来获取 `DOM` 节点中的 `data-*` 属性的数据，和储存跟 `DOM` 相关的数据。

读 Zepto 源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## GitBook

《[reading-zepto](https://yeyuqiudeng.gitbooks.io/reading-zepto/content/)》

## 内部方法

### attributeData

```javascript
var data = {}, dataAttr = $.fn.data, camelize = $.camelCase,
    exp = $.expando = 'Zepto' + (+new Date()), emptyArray = []
function attributeData(node) {
  var store = {}
  $.each(node.attributes || emptyArray, function(i, attr){
    if (attr.name.indexOf('data-') == 0)
      store[camelize(attr.name.replace('data-', ''))] =
        $.zepto.deserializeValue(attr.value)
  })
  return store
}
```

这个方法用来获取给定 `node` 中所有 `data-*` 属性的值，并储存到 `store` 对象中。

`node.attributes` 获取到的是节点的所有属性，因此在遍历的时候，需要判断属性名是否以 `data-` 开头。

在存储的时候，将属性名的 `data-` 去掉，剩余部分转换成驼峰式，作为 `store` 对象的 `key` 。

在 `DOM` 中的属性值都为字符串格式，为方便操作，调用 `deserializeValue` 方法，转换成对应的数据类型，关于这个方法的具体分析，请看 《[读Zepto源码之属性操作](https://github.com/yeyuqiudeng/reading-zepto/blob/6fb60c6a6ca1cf4f6846c32883774b5ba0f7de45/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B1%9E%E6%80%A7%E6%93%8D%E4%BD%9C.md#deserializevalue)》

### setData

```javascript
function setData(node, name, value) {
  var id = node[exp] || (node[exp] = ++$.uuid),
      store = data[id] || (data[id] = attributeData(node))
  if (name !== undefined) store[camelize(name)] = value
  return store
}
```

更多时候，储存数据不需要写在 `DOM` 中，只需要储存在内存中即可。而且读取 `DOM` 的成本非常高。

`setData` 方法会将对应 `DOM` 的数据储存在 `store` 对象中。

```javascript
var id = node[exp] || (node[exp] = ++$.uuid)
```

首先读取 `node` 的 `exp` 属性，从前面可以看到 `exp` 是一个 `Zepto` 加上时间戳的字符串，以确保属性名的唯一性，避免覆盖用户自定义的属性，如果 `node` 尚未打上 `exp` 标记，表明这个节点并没有缓存的数据，则设置节点的 `exp` 属性。

```javascript
store = data[id] || (data[id] = attributeData(node))
```

从 `data` 中获取节点的之前缓存的数据，如果之前没有缓存数据，则调用 `attributeData` 方法，获取节点上所有以 `data-` 开头的属性值，缓存到 `data` 对象中。

```javascript
store[camelize(name)] = value
```

最后，设置需要缓存的值。

### getData

```javascript
function getData(node, name) {
  var id = node[exp], store = id && data[id]
  if (name === undefined) return store || setData(node)
  else {
    if (store) {
      if (name in store) return store[name]
      var camelName = camelize(name)
      if (camelName in store) return store[camelName]
    }
    return dataAttr.call($(node), name)
  }
}
```

获取 `node` 节点上指定的缓存值。

```javascript
if (name === undefined) return store || setData(node)
```

如果没有指定属性名，则将节点对应的缓存全部返回，如果缓存为空，则调用 `setData` 方法，返回 `node` 节点上所有的 `data-` 开头的属性值。

```javascript
if (name in store) return store[name]
```

如果指定的 `name` 在缓存 `store` 中，则将结果返回。

```javascript
var camelName = camelize(name)
if (camelName in store) return store[camelName]
```

否则，将指定的 `name` 转换成驼峰式，再从缓存 `store` 中查找，将找到的结果返回。这是兼容 `camel-name` 这样的参数形式，提供更灵活的 `API` 。

如果缓存中都没找到，则回退到用 `$.fn.data` 查找，其实就是查找 `data-` 属性上的值，这个方法后面会分析到。

## DOM方法

### .data()

```javascript
$.fn.data = function(name, value) {
  return value === undefined ?
    $.isPlainObject(name) ?
    this.each(function(i, node){
    $.each(name, function(key, value){ setData(node, key, value) })
  }) :
  (0 in this ? getData(this[0], name) : undefined) :
  this.each(function(){ setData(this, name, value) })
}
```

`data` 方法可以设置或者获取对应 `node` 节点的缓存数据，最终分别调用的是 `setData` 和 `getData` 方法。

分析这段代码，照例还是将三元表达式一个一个拆解，来看看都做了什么事情。

```javascript
value === undefined ? 三元表达式 : this.each(function(){ setData(this, name, value) })
```

先看第一层，当有传递 `name` 和 `value` 时，表明是设置缓存，遍历所有元素，分别调用 `setData` 方法设置缓存。

```javascript
$.isPlainObject(name) ?
    this.each(function(i, node){
    $.each(name, function(key, value){ setData(node, key, value) })
  }) : 三元表达式
```

`data` 的第一个参数还支持对象的传值，例如 `$(el).data({key1: 'value1'})` 。如果是对象，则对象里的属性为需要设置的缓存名，值为缓存值。

因此，也遍历所有元素，调用 `setData` 设置缓存。

```javascript
0 in this ? getData(this[0], name) : undefined
```

最后，判断集合是否不为空（ `0 in this` ）， 如果为空，则直接返回 `undefined` ，否则，调用 `getData` ，返回第一个元素节点对应的 `name` 的缓存。

### removeData

```javascript
$.fn.removeData = function(names) {
  if (typeof names == 'string') names = names.split(/\s+/)
  return this.each(function(){
    var id = this[exp], store = id && data[id]
    if (store) $.each(names || store, function(key){
      delete store[names ? camelize(this) : key]
    })
  })
}
```

`removeData` 用来删除缓存的数据，如果没有传递参数，则全部清空，如果有传递参数，则只删除指定的数据。

`names` 可以为数组，指定需要删除的一组数据，也可以为以空格分割的字符串。

```javascript
if (typeof names == 'string') names = names.split(/\s+/)
```

如果检测到 `names` 为字符串，则先将字符串转换成数组。

```javascript
return this.each(function(){
  var id = this[exp], store = id && data[id]
 ...
})
```

遍历元素，对所有的元素都进行删除操作，找出和元素对应的缓存 `store` 。

```javascript
if (store) $.each(names || store, function(key){
  delete store[names ? camelize(this) : key]
})
```

如果 `names` 存在，则删除指定的数据，否则将 `store` 缓存的数据全部删除。

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
19. [读Zepto源码之IOS3模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BIOS3%E6%A8%A1%E5%9D%97.md)
20. [读Zepto源码之Fx模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BFx%E6%A8%A1%E5%9D%97.md)
21. [读Zepto源码之fx_methods模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8Bfx_methods%E6%A8%A1%E5%9D%97.md)
22. [读Zepto源码之Stack模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BStack%E6%A8%A1%E5%9D%97.md)
23. [读Zepto源码之Form模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BForm%E6%A8%A1%E5%9D%97.md)

### 附文

- [译：怎样处理 Safari 移动端对图片资源的限制](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E9%99%84%EF%BC%9A%E6%80%8E%E6%A0%B7%E5%A4%84%E7%90%86%20Safari%20%E7%A7%BB%E5%8A%A8%E7%AB%AF%E5%AF%B9%E5%9B%BE%E7%89%87%E8%B5%84%E6%BA%90%E7%9A%84%E9%99%90%E5%88%B6.md)

## 参考

* [Zepto中数据缓存原理与实现](https://segmentfault.com/a/1190000011443975)
* [Element.attributes](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/attributes)

## License

[署名-非商业性使用-禁止演绎 4.0 国际 (CC BY-NC-ND 4.0)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://raw.githubusercontent.com/yeyuqiudeng/resource/master/images/qrcode_front-end-article.jpg) 

作者：对角另一面