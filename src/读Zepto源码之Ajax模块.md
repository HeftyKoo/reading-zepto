# 读Zepto源码之Ajax模块

`Ajax` 模块也是经常会用到的模块，`Ajax` 模块中包含了 `jsonp` 的现实，和 `XMLHttpRequest` 的封装。 

读 Zepto 源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## ajax的事件触发顺序

`zepto` 针对 `ajax` 的发送过程，定义了以下几个事件，正常情况下的触发顺序如下：

1.  `ajaxstart` : `XMLHttpRequest` 实例化前触发
2.  `ajaxBeforeSend`： 发送 `ajax` 请求前触发
3.  `ajaxSend` : 发送 `ajax` 请求时触发
4.  `ajaxSuccess` / `ajaxError` : 请求成功/失败时触发
5.  `ajaxComplete`： 请求完成（无论成功还是失败）时触发
6.  `ajaxStop`: 请求完成后触发，这个事件在 `ajaxComplete` 后触发。

### ajax 方法的参数解释

现在还没有讲到 `ajax` 方法，之所以要将参数提前，是因为后面的内容，不时会用到相关的参数，所以一开始先将参数解释清楚。

* `type`： `HTTP` 请求的类型；
* `url`: 请求的路径；
* `data`： 请求参数；
* `processData`: 是否需要将非 `GET` 请求的参数转换成字符串，默认为 `true` ，即默认转换成字符串；
* `contentType`: 设置 `Content-Type` 请求头；
* `mineType` ： 覆盖响应的 `MIME` 类型，可以是 `json`、 `jsonp`、 `script`、 `xml`、 `html`、 或者 `text`；
* `jsonp`:  `jsonp` 请求时，携带参数的参数名，默认为 `callback`；
* `jsonpCallback`： `jsonp` 请求时，响应成功时，执行的回调函数名，默认由 `zepto` 管理；
* `timeout`: 超时时间，默认为 `0`；
* `headers`：设置 `HTTP` 请求头；
* `async`： 是否为同步请求，默认为 `false`；
* `global`： 是否触发全局 `ajax` 事件，默认为 `true`；
* `context`： 执行回调时（如 `jsonpCallbak`）时的上下文环境，默认为 `window`。
* `traditional`: 是否使用传统的浅层序列化方式序列化 `data` 参数，默认为 `false`，例如有 `data` 为 `{p1:'test1', p2: {nested: 'test2'}` ，在 `traditional` 为 `false` 时，会序列化成 `p1=test1&p2[nested]=test2`， 在为 `true` 时，会序列化成 `p1=test&p2=[object+object]`；
* `xhrFields`：`xhr` 的配置；
* `cache`：是否允许浏览器缓存 `GET` 请求，默认为 `false`；
* `username`：需要认证的 `HTTP` 请求的用户名；
* `password`： 需要认证的 `HTTP` 请求的密码；
* `dataFilter`： 对响应数据进行过滤；
* `xhr`： `XMLHttpRequest` 实例，默认用 `XMLHttpRequest` 生成；
* `accepts`：从服务器请求的 `MIME` 类型；
* `beforeSend`： 请求发出前调用的函数；
* `success`: 请求成功后调用的函数；
* `error`： 请求出错时调用的函数；
* `complete`： 请求完成时调用的函数，无论请求是失败还是成功。


正在请求的 `ajax` 数量，初始数为 `0`。

## 内部方法

### triggerAndReturn

```javascript
function triggerAndReturn(context, eventName, data) {
  var event = $.Event(eventName)
  $(context).trigger(event, data)
  return !event.isDefaultPrevented()
}
```

`triggerAndReturn` 用来触发一个事件，并且如果该事件禁止浏览器默认事件时，返回 `false`。

参数 `context` 为上下文，`eventName` 为事件名，`data` 为数据。

该方法内部调用了 `Event` 模块的 `trigger` 方法，具体分析见《[读Zepto源码之Event模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BEvent%E6%A8%A1%E5%9D%97.md#trigger)》。

### triggerGlobal

```javascript
function triggerGlobal(settings, context, eventName, data) {
  if (settings.global) return triggerAndReturn(context || document, eventName, data)
}
```

触发全局事件

`settings` 为 `ajax` 配置，`context` 为指定的上下文对象，`eventName` 为事件名，`data` 为数据。

`triggerGlobal` 内部调用的是 `triggerAndReturn` 方法，如果有指定上下文对象，则在指定的上下文对象上触发，否则在 `document` 上触发。

### ajaxStart

```javascript
function ajaxStart(settings) {
  if (settings.global && $.active++ === 0) triggerGlobal(settings, null, 'ajaxStart')
}
```

触发全局的 `ajaxStart` 事件。

如果 `global` 设置为 `true`，则 `$.active` 的值增加1。

如果 `global` 为 `true` ，并且 `$.active` 在更新前的数量为 `0`，则触发全局的 `ajaxStart` 事件。

### ajaxStop

```javascript
function ajaxStop(settings) {
  if (settings.global && !(--$.active)) triggerGlobal(settings, null, 'ajaxStop')
}
```

触发全局 `ajaxStop` 事件。

如果 `global` 为 `true` ，则将 `$.active` 的数量减少 `1`。如果 `$.active` 的数量减少至 `0`，即没有在执行中的 `ajax` 请求时，触发全局的 `ajaxStop` 事件。

### ajaxBeforeSend

```javascript
function ajaxBeforeSend(xhr, settings) {
  var context = settings.context
  if (settings.beforeSend.call(context, xhr, settings) === false ||
      triggerGlobal(settings, context, 'ajaxBeforeSend', [xhr, settings]) === false)
    return false

  triggerGlobal(settings, context, 'ajaxSend', [xhr, settings])
}
```

`ajaxBeforeSend` 方法，触发 `ajaxBeforeSend` 事件和 `ajaxSend` 事件。

这两个事件很相似，只不过 `ajaxBeforedSend` 事件可以通过外界的配置来取消事件的触发。

在触发 `ajaxBeforeSend` 事件之前，会调用配置中的 `beforeSend` 方法，如果 `befoeSend` 方法返回的为 `false`时，则取消触发 `ajaxBeforeSend` 事件。

否则触发 `ajaxBeforeSend` 事件，并且将 `xhr` 事件，和配置 `settings` 作为事件携带的数据。

注意这里很巧妙地使用了 `||` 进行断路。

如果 `beforeSend` 返回的为 `false` 或者触发`ajaxBeforeSend` 事件的方法 `triggerGlobal` 返回的为 `false`，也即取消了浏览器的默认行为，则 `ajaxBeforeSend` 方法返回 `false`，中止后续的执行。

否则在触发完 `ajaxBeforeSend` 事件后，触发 `ajaxSend` 事件。

### ajaxComplete

```javascript
function ajaxComplete(status, xhr, settings) {
  var context = settings.context
  settings.complete.call(context, xhr, status)
  triggerGlobal(settings, context, 'ajaxComplete', [xhr, settings])
  ajaxStop(settings)
}
```

触发 `ajaxComplete` 事件。

在触发 `ajaxComplete` 事件前，调用配置中的 `complete` 方法，将 `xhr` 实例和当前的状态 `state` 作为回调函数的参数。在触发完 `ajaxComplete` 事件后，调用 `ajaxStop` 方法，触发 `ajaxStop` 事件。

### ajaxSuccess

```javascript
function ajaxSuccess(data, xhr, settings, deferred) {
  var context = settings.context, status = 'success'
  settings.success.call(context, data, status, xhr)
  if (deferred) deferred.resolveWith(context, [data, status, xhr])
  triggerGlobal(settings, context, 'ajaxSuccess', [xhr, settings, data])
  ajaxComplete(status, xhr, settings)
}
```

触发 `ajaxSucess` 方法。

在触发 `ajaxSuccess` 事件前，先调用配置中的 `success` 方法，将 `ajax` 返回的数据 `data` 和当前状态 `status` 及 `xhr` 作为回调函数的参数。

如果 `deferred` 存在，则调用 `resoveWith` 的方法，因为 `deferred` 对象，因此在使用 `ajax` 的时候，可以使用 `promise` 风格的调用。关于 `deferred` ，见 《[读Zepto源码之Deferred模块](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8BDeferred%E6%A8%A1%E5%9D%97.md)》的分析。 

在触发完 `ajaxSuccess` 事件后，继续调用 `ajaxComplete` 方法，触发 `ajaxComplete` 事件。

### ajaxError

```javascript
function ajaxError(error, type, xhr, settings, deferred) {
  var context = settings.context
  settings.error.call(context, xhr, type, error)
  if (deferred) deferred.rejectWith(context, [xhr, type, error])
  triggerGlobal(settings, context, 'ajaxError', [xhr, settings, error || type])
  ajaxComplete(type, xhr, settings)
}
```

触发 `ajaxError` 事件，错误的类型可以为 `timeout`、`error`、 `abort`、 `parsererror`。

在触发事件前，调用配置中的 `error` 方法，将 `xhr` 实例，错误类型 `type` 和 `error` 对象作为回调函数的参数。

随后调用 `ajaxComplete` 方法，触发 `ajaxComplete` 事件。因此，`ajaxComplete` 事件无论成功还是失败都会触发。

### empty

```javascript
function empty() {}
```

空函数，用来作为回调函数配置的初始值。这样的好处是在执行回调函数时，不需要每次都判断回调函数是否存在。

### ajaxDataFilter

```javascript
function ajaxDataFilter(data, type, settings) {
  if (settings.dataFilter == empty) return data
  var context = settings.context
  return settings.dataFilter.call(context, data, type)
}
```

主要用来过滤请求成功后的响应数据。

如果配置中的 `dataFilter` 属性为初始值 `empty`，则将原始数据返回。

如果有配置 `dataFilter`，则调用配置的回调方法，将数据 `data` 和数据类型 `type` 作为回调的参数，再将执行的结果返回。

### mimeToDataType

```javascript
var htmlType = 'text/html',
    jsonType = 'application/json',
    scriptTypeRE = /^(?:text|application)\/javascript/i,
    xmlTypeRE = /^(?:text|application)\/xml/i,
function mimeToDataType(mime) {
  if (mime) mime = mime.split(';', 2)[0]
  return mime && ( mime == htmlType ? 'html' :
                  mime == jsonType ? 'json' :
                  scriptTypeRE.test(mime) ? 'script' :
                  xmlTypeRE.test(mime) && 'xml' ) || 'text'
}
```

返回 `dataType` 的类型。

先看看这个函数中使用到的几个正则表达式，`scriptTypeRE` 匹配的是 `text/javascript` 或者 `application/javascript`， `xmlTypeRE` 匹配的是 `text/xml` 或者 `application/xml`， 都还比较简单，不作过多的解释了。

`Content-Type` 的值的形式如下 `text/html; charset=utf-8`， 所以如果参数 `mime` 存在，则用 `;` 分割，取第一项，这里是 `text/html`，即为包含类型的字符串。

接下来是针对 `html` 、`json`、 `script` 和  `xml` 用对应的正则进行匹配，匹配成功，返回对应的类型值，如果都不匹配，则返回 `text`。

### appendQuery

```javascript
function appendQuery(url, query) {
  if (query == '') return url
  return (url + '&' + query).replace(/[&?]{1,2}/, '?')
}
```

 向 `url` 追加参数。

如果 `query` 为空，则将原 `url` 返回。

如果 `query` 不为空，则用 `&` 拼接 `query`。

最后调用 `replace`，将 `&&` 、 `?&` ，`&?`  或 `??` 替换成 `?`。

拼接出来的 `url` 的形式如 `url?key=value&key2=value`

### parseArguments

```javascript
function parseArguments(url, data, success, dataType) {
  if ($.isFunction(data)) dataType = success, success = data, data = undefined
  if (!$.isFunction(success)) dataType = success, success = undefined
  return {
    url: url
    , data: data
    , success: success
    , dataType: dataType
  }
}
```

 这个方法是用来格式化参数的，`Ajax` 模块定义了一些便捷的调用方法，这些调用方法不需要传递 `option`，某些必填值已经采用了默认传递的方式，这些方法中有些参数是可以不需要传递的，这个方法就是来用判读那些参数有传递，那些没有传递，然后再将参数拼接成 `ajax` 所需要的 `options` 对象。

### serialize

```javascript
function serialize(params, obj, traditional, scope){
  var type, array = $.isArray(obj), hash = $.isPlainObject(obj)
  $.each(obj, function(key, value) {
    type = $.type(value)
    if (scope) key = traditional ? scope :
    scope + '[' + (hash || type == 'object' || type == 'array' ? key : '') + ']'
    // handle data in serializeArray() format
    if (!scope && array) params.add(value.name, value.value)
    // recurse into nested objects
    else if (type == "array" || (!traditional && type == "object"))
      serialize(params, value, traditional, key)
    else params.add(key, value)
  })
}
```

序列化参数。

要了解这个函数，需要了解 `traditional` 参数的作用，这个参数表示是否开启以传统的浅层序列化方式来进行序列化，具体的示例见上文参数解释部分。

如果参数 `obj` 的为数组，则 `array` 为 `true`， 如果为纯粹对象，则 `hash` 为 `true`。 `$.isArray` 和 `$.isPlainObject` 的源码分析见《[读Zepto源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/456627c2e8199ac1d043af51177d6159fddd39bc/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md#isplainobject)》。

遍历需要序列化的对象 `obj`，判断 `value` 的类型 `type`， 这个 `type` 后面会用到。

`scope` 是记录深层嵌套时的 `key` 值，这个 `key` 值受 `traditional` 的影响。

如果 `traditional` 为 `true` ，则 `key` 为原始的 `scope` 值，即对象第一层的 `key` 值。

否则，用 `[]` 拼接当前循环中的 `key` ，最终的 `key` 值会是这种形式 `scope[key][key2]...`

如果 `obj` 为数组，并且 `scope` 不存在，即为第一层，直接调用 `params.add` 方法，这个方法后面会分析到。

否则如果  `value` 的类型为数组或者非传统序列化方式下为对象，则递归调用 `serialize` 方法，用来处理 `key` 。

其他情况调用 `params.add` 方法。

### serializeData

```javascript
function serializeData(options) {
  if (options.processData && options.data && $.type(options.data) != "string")
    options.data = $.param(options.data, options.traditional)
  if (options.data && (!options.type || options.type.toUpperCase() == 'GET' || 'jsonp' == options.dataType))
    options.url = appendQuery(options.url, options.data), options.data = undefined
}
```

序列化参数。

如果 `processData` 为 `true` ，并且参数 `data` 不为字符串，则调用 `$.params` 方法序列化参数。 `$.params` 方法后面会讲到。

如果为 `GET` 请求或者为 `jsonp` ，则调用 `appendQuery` ，将参数拼接到请求地址后面。



## 系列文章

1. [读Zepto源码之代码结构](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E4%BB%A3%E7%A0%81%E7%BB%93%E6%9E%84.md)
2. [读 Zepto 源码之内部方法](https://github.com/yeyuqiudeng/reading-zepto/blob/master/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%86%85%E9%83%A8%E6%96%B9%E6%B3%95.md)
3. [读Zepto源码之工具函数](https://github.com/yeyuqiudeng/reading-zepto/blob/a4d6ad99c57047beae2b652b4d2cbb380599a524/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md)
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



## 参考

* [Zepto源码分析-ajax模块](http://www.cnblogs.com/mominger/p/4398982.html)
* [读zepto源码（3) ajax](http://ysha.me/2016/07/19/07-19-%E8%AF%BBzepto%E6%BA%90%E7%A0%81%EF%BC%883%EF%BC%89ajax/)
* [你真的会使用XMLHttpRequest吗？](https://segmentfault.com/a/1190000004322487)
* [原来你是这样的 jsonp(原理与具体实现细节)](https://juejin.im/post/593d7f0a128fe1006aea235f)

## License

[![License: CC BY-NC-ND 4.0](https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://user-gold-cdn.xitu.io/2017/5/30/76626b0be42083d36b36f4a117dc1873) 

作者：对角另一面

