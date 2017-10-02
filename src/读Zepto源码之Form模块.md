# 读Zepto源码之Form模块

`Form` 模块处理的是表单提交。表单提交包含两部分，一部分是格式化表单数据，另一部分是触发 `submit` 按钮，提交表单。

读 Zepto 源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## GitBook

《[reading-zepto](https://yeyuqiudeng.gitbooks.io/reading-zepto/content/)》

## .serializeArray()

```javascript
$.fn.serializeArray = function() {
  var name, type, result = [],
      add = function(value) {
        if (value.forEach) return value.forEach(add)
        result.push({ name: name, value: value })
      }
  if (this[0]) $.each(this[0].elements, function(_, field){
    type = field.type, name = field.name
    if (name && field.nodeName.toLowerCase() != 'fieldset' &&
        !field.disabled && type != 'submit' && type != 'reset' && type != 'button' && type != 'file' &&
        ((type != 'radio' && type != 'checkbox') || field.checked))
      add($(field).val())
  })
  return result
}
```

`serializeArray` 是格式化部分的核心方法，后面的 `serialize` 方法内部调用的也是 `serializeArray` 方法。

`serializeArray` 最终返回的结果是一个数组，每个数组项为包含 `name` 和 `value` 属性的对象。其中 `name` 为表单元素的 `name` 属性值。

### add函数

```javascript
add = function(value) {
  if (value.forEach) return value.forEach(add)
  result.push({ name: name, value: value })
}
```

表单的值交由 `add` 函数处理，如果值为数组（支持 `forEach` ） 方法，则调用 `forEach` 遍历，继续由 `add` 函数处理。否则将结果存入数组 `result` 中。最后返回的结果也是这个  `result`。

### 遍历表单元素

```javascript
if (this[0]) $.each(this[0].elements, function(_, field){
  type = field.type, name = field.name
  if (name && field.nodeName.toLowerCase() != 'fieldset' &&
      !field.disabled && type != 'submit' && type != 'reset' && type != 'button' && type != 'file' &&
      ((type != 'radio' && type != 'checkbox') || field.checked))
    add($(field).val())
})
```

如果集合中有多个表单，则只处理第一个表单的表单元素。`this[0].elements`  用来获取所有第一个表单所有的表单元素。

`type` 为表单类型，`name` 为表单元素的 `name` 属性值。

这一大段代码的关键在 `if` 中的条件判断，其实是将一些无关的表单元素排除，只处理符合条件的表单元素。

以下一个条件一个条件来分析：

* `field.nodeName.toLowerCase() != 'fieldset'` 排除 `fueldset` 元素；
* `!field.disabled` 排除禁用的表单，已经禁用了，肯定是没有值需要提交的了；
* `type != 'submit'` 排除确定按钮；
* `type != 'reset'` 排除重置按钮；
* `type != 'button'` 排除按钮；
* `type != 'file'` 排除文件选择控件；
* `((type != 'radio' && type != 'checkbox') || field.checked))` 如果是 `radio` 或 `checkbox` 时，则必须要选中，这个也很好理解，如果没有选中，也不会有值需要处理。

然后调用 `add` 方法，将表单元素的值获取到交由其处理。

## .serialize()

```javascript
$.fn.serialize = function(){
  var result = []
  this.serializeArray().forEach(function(elm){
    result.push(encodeURIComponent(elm.name) + '=' + encodeURIComponent(elm.value))
  })
  return result.join('&')
}
```

表单元素处理完成后，最终是要拼成如 `name1=value1&name2=value2&...` 的形式，`serialize` 方法要做的就是这部分事情。

这里对 `serizlizeArray` 返回的数组再做进一步的处理，首先用 `encodeURIComponent` 序列化 `name` 和 `value` 的值，并用 `=` 号拼接成字符串，存进新的数组中，最后调用 `join` 方法，用 `&` 将各项拼接起来。

## .submit()

```javascript
$.fn.submit = function(callback) {
  if (0 in arguments) this.bind('submit', callback)
  else if (this.length) {
    var event = $.Event('submit')
    this.eq(0).trigger(event)
    if (!event.isDefaultPrevented()) this.get(0).submit()
  }
  return this
}
```

处理完数据，接下来该到提交了。

```javascript
if (0 in arguments) this.bind('submit', callback)
```

如果有传递回调函数 `callback` ，则在表单上绑定 `submit` 事件，以 `callback` 作为事件的回调。

```javascript
else if (this.length) {
  var event = $.Event('submit')
  this.eq(0).trigger(event)
  if (!event.isDefaultPrevented()) this.get(0).submit()
}
```

否则手动绑定 `submit` 事件，如果没有阻止浏览器的默认事件，则在第一个表单上触发 `submit` ，提交表单。

注意 `eq` 和 `get` 的区别， `eq` 返回的是 `Zepto` 对象，而 `get` 返回的是 `DOM` 元素。

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

### 附文

- [译：怎样处理 Safari 移动端对图片资源的限制](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E9%99%84%EF%BC%9A%E6%80%8E%E6%A0%B7%E5%A4%84%E7%90%86%20Safari%20%E7%A7%BB%E5%8A%A8%E7%AB%AF%E5%AF%B9%E5%9B%BE%E7%89%87%E8%B5%84%E6%BA%90%E7%9A%84%E9%99%90%E5%88%B6.md)

## 参考

- [zepto源码分析之form模块](https://juejin.im/post/59d07c03f265da0668761e82?utm_source=gold_browser_extension)
- [HTMLFormElement.elements](https://developer.mozilla.org/en-US/docs/Web/API/HTMLFormElement/elements)

## License

[署名-非商业性使用-禁止演绎 4.0 国际 (CC BY-NC-ND 4.0)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://raw.githubusercontent.com/yeyuqiudeng/resource/master/images/qrcode_front-end-article.jpg) 

作者：对角另一面

