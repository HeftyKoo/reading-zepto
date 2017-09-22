# 读Zepto源码之Ajax模块

`Ajax` 模块也是经常会用到的模块，`Ajax` 模块中包含了 `jsonp` 的现实，和 `XMLHttpRequest` 的封装。 

读 Zepto 源码系列文章已经放到了github上，欢迎star: [reading-zepto](https://github.com/yeyuqiudeng/reading-zepto)

## 源码版本

本文阅读的源码为 [zepto1.2.0](https://github.com/madrobby/zepto/tree/v1.2.0)

## GitBook

《[reading-zepto](https://yeyuqiudeng.gitbooks.io/reading-zepto/content/)》

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
* `jsonp`:  `jsonp` 请求时，携带回调函数名的参数名，默认为 `callback`；
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
* `xhr`： `XMLHttpRequest` 实例，默认用 `new XMLHttpRequest()` 生成；
* `accepts`：从服务器请求的 `MIME` 类型；
* `beforeSend`： 请求发出前调用的函数；
* `success`: 请求成功后调用的函数；
* `error`： 请求出错时调用的函数；
* `complete`： 请求完成时调用的函数，无论请求是失败还是成功。


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

在触发 `ajaxBeforeSend` 事件之前，会调用配置中的 `beforeSend` 方法，如果 `befoeSend` 方法返回的为 `false`时，则取消触发 `ajaxBeforeSend` 事件，并且会取消后续 `ajax` 请求的发送，后面会讲到。

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

先看看这个函数中使用到的几个正则表达式，`scriptTypeRE` 匹配的是 `text/javascript` 或者 `application/javascript`， `xmlTypeRE` 匹配的是 `text/xml` 或者 `application/xml`， 都还比较简单，不作过多的解释。

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

## 对外接口

### $.active 

```javascript
$.active = 0
```
正在请求的 `ajax` 数量，初始时为 `0`。

### $.ajaxSettings

```javascript
$.ajaxSettings = {
  // Default type of request
  type: 'GET',
  // Callback that is executed before request
  beforeSend: empty,
  // Callback that is executed if the request succeeds
  success: empty,
  // Callback that is executed the the server drops error
  error: empty,
  // Callback that is executed on request complete (both: error and success)
  complete: empty,
  // The context for the callbacks
  context: null,
  // Whether to trigger "global" Ajax events
  global: true,
  // Transport
  xhr: function () {
    return new window.XMLHttpRequest()
  },
  // MIME types mapping
  // IIS returns Javascript as "application/x-javascript"
  accepts: {
    script: 'text/javascript, application/javascript, application/x-javascript',
    json:   jsonType,
    xml:    'application/xml, text/xml',
    html:   htmlType,
    text:   'text/plain'
  },
  // Whether the request is to another domain
  crossDomain: false,
  // Default timeout
  timeout: 0,
  // Whether data should be serialized to string
  processData: true,
  // Whether the browser should be allowed to cache GET responses
  cache: true,
  //Used to handle the raw response data of XMLHttpRequest.
  //This is a pre-filtering function to sanitize the response.
  //The sanitized response should be returned
  dataFilter: empty
}
```

`ajax` 默认配置，这些是 `zepto` 的默认值，在使用时，可以更改成自己需要的配置。

### $.param

```javascript
var escape = encodeURIComponent
$.param = function(obj, traditional){
  var params = []
  params.add = function(key, value) {
    if ($.isFunction(value)) value = value()
    if (value == null) value = ""
    this.push(escape(key) + '=' + escape(value))
  }
  serialize(params, obj, traditional)
  return params.join('&').replace(/%20/g, '+')
}
```

`param` 方法用来序列化参数，内部调用的是 `serialize` 方法，并且在容器 `params` 上定义了一个 `add` 方法，供 `serialize` 调用。

`add` 方法比较简单，首先判断值 `value` 是否为 `function` ，如果是，则通过调用函数来取值，如果为 `null` 或者 `undefined` ，则 `value` 赋值为空字符串。

然后将 `key` 和 `value` 用 `encodeURIComponent` 编码，用 `=` 号连接起来。

接着便是简单的调用 `serialize` 方法。

最后将容器中的数据用 `&` 连接起来，并且将空格替换成 `+` 号。

### $.ajaxJSONP

```javascript
var jsonpID = +new Date()
$.ajaxJSONP = function(options, deferred){
  if (!('type' in options)) return $.ajax(options)

  var _callbackName = options.jsonpCallback,
      callbackName = ($.isFunction(_callbackName) ?
                      _callbackName() : _callbackName) || ('Zepto' + (jsonpID++)),
      script = document.createElement('script'),
      originalCallback = window[callbackName],
      responseData,
      abort = function(errorType) {
        $(script).triggerHandler('error', errorType || 'abort')
      },
      xhr = { abort: abort }, abortTimeout

  if (deferred) deferred.promise(xhr)

  $(script).on('load error', function(e, errorType){
    clearTimeout(abortTimeout)
    $(script).off().remove()

    if (e.type == 'error' || !responseData) {
      ajaxError(null, errorType || 'error', xhr, options, deferred)
    } else {
      ajaxSuccess(responseData[0], xhr, options, deferred)
    }

    window[callbackName] = originalCallback
    if (responseData && $.isFunction(originalCallback))
      originalCallback(responseData[0])

    originalCallback = responseData = undefined
  })

  if (ajaxBeforeSend(xhr, options) === false) {
    abort('abort')
    return xhr
  }

  window[callbackName] = function(){
    responseData = arguments
  }

  script.src = options.url.replace(/\?(.+)=\?/, '?$1=' + callbackName)
  document.head.appendChild(script)

  if (options.timeout > 0) abortTimeout = setTimeout(function(){
    abort('timeout')
  }, options.timeout)

  return xhr
}
```

在分析源码之前，先了解一下 `jsonp` 的原理。

`jsonp` 实现跨域其实是利用了 `script` 可以请求跨域资源的特点，所以实现 `jsonp` 的基本步骤就是向页面动态插入一个 `script` 标签，在请求地址上带上需要传递的参数，后端再将数据返回，前端调用回调函数进行解释。

所以 `jsonp` 本质上是一个 `GET` 请求，因为链接的长度有限制，因此请求所携带的参数的长度也会有限制。

#### 一些变量的定义

```javascript
if (!('type' in options)) return $.ajax(options)

var _callbackName = options.jsonpCallback,
    callbackName = ($.isFunction(_callbackName) ?
                    _callbackName() : _callbackName) || ('Zepto' + (jsonpID++)),
    script = document.createElement('script'),
    originalCallback = window[callbackName],
    responseData,
    abort = function(errorType) {
      $(script).triggerHandler('error', errorType || 'abort')
    },
    xhr = { abort: abort }, abortTimeout

if (deferred) deferred.promise(xhr)
```

如果配置中的请求类型没有定义，则直接调用 `$.ajax` 方法，这个方法是整个模块的核心，后面会讲到。 `jsonp` 请求的 `type` 必须为 `jsonp` 。

私有变量用来临时存放配置中的 `jsonpCallback` ，即 `jsonp` 请求成功后执行的回调函数名，该配置可以为 `function` 类型。

`callbackName` 是根据配置得出的回调函数名。如果 `_callbackName` 为 `function` ，则以执行的结果作为回调函数名，如果 `_callbackName` 没有配置，则用 `Zepto` + `时间戳` 作为回调函数名，时间戳初始化后，采用自增的方式来实现函数名的唯一性。

`script` 用来保存创建的 `script` 节点。

`originalCallback` 用来储存原始的回调函数。

`responseData` 为响应的数据。

`abort` 函数用来中止 `jsonp` 请求，实质上是触发了 `error` 事件。

`xhr` 对象只有 `abort` 方法，如果存在 `deferred` 对象，则调用 `promise` 方法在 `xhr` 对象的基础上生成一个 `promise` 对象。

`abortTimeout` 用来指定超时时间。

#### beforeSend

```javascript
if (ajaxBeforeSend(xhr, options) === false) {
  abort('abort')
  return xhr
}
```

在发送 `jsonp` 请求前，会调用 `ajaxBeforeSend` 方法，如果返回的为 `false`，则中止 `jsonp` 请求的发送。

#### 发送请求

```javascript
window[callbackName] = function(){
  responseData = arguments
}

script.src = options.url.replace(/\?(.+)=\?/, '?$1=' + callbackName)
document.head.appendChild(script)
```

发送请求前，重写了 `window[callbackName]` 函数，将 `arguments` 赋值给 `responseData`， 这个函数会在后端返回的 `js` 代码中执行，这样 `responseData` 就可以获取得到数据了。

接下来，将 `url` 的`=?` 占位符，替换成回调函数名，最后将 `script` 插入到页面中，发送请求。

#### 请求超时

```javascript
if (options.timeout > 0) abortTimeout = setTimeout(function(){
  abort('timeout')
}, options.timeout)
```

 如果有设置超时时间，则在请求超时时，触发错误事件。

#### 请求成功或失败

```javascript
$(script).on('load error', function(e, errorType){
  clearTimeout(abortTimeout)
  $(script).off().remove()

  if (e.type == 'error' || !responseData) {
    ajaxError(null, errorType || 'error', xhr, options, deferred)
  } else {
    ajaxSuccess(responseData[0], xhr, options, deferred)
  }

  window[callbackName] = originalCallback
  if (responseData && $.isFunction(originalCallback))
    originalCallback(responseData[0])

  originalCallback = responseData = undefined
})
```

在请求成功或者失败时，先清除请求超时定时器，避免触发超时错误，再将插入页面的 `script` 从页面上删除，因为数据已经获取到，不再需要这个 `script` 了。注意在删除 `script` 前，调用了 `off` 方法，将 `script` 上的事件都移除了。

如果请求出错，则调用 `ajaxError` 方法。

如果请求成功，则调用 `ajaxSuccess` 方法。

之前我们把 `window[callbackName]` 重写掉了，目的是为了获取到数据，现在再重新将原来的回调函数赋值回去，在获取到数据后，如果 `originalCallback` 有定义，并且为函数，则将数据作为参数传递进去，执行。

最后将数据和临时函数 `originalCallback` 清理。

### $.ajax

`$.ajax` 方法是整个模块的核心，代码太长，就不全部贴在这里了，下面一部分一部分来分析。

#### 处理默认配置

```javascript
var settings = $.extend({}, options || {}),
    deferred = $.Deferred && $.Deferred(),
    urlAnchor, hashIndex
for (key in $.ajaxSettings) if (settings[key] === undefined) settings[key] = $.ajaxSettings[key]
ajaxStart(settings)
```

`settings` 为所传递配置的副本。

`deferred` 为 `deferred` 对象。

`urlAnchor` 为浏览器解释的路径，会用来判断是否跨域，后面会讲到。

`hashIndex` 为路径中 `hash` 的索引。

用 `for ... in` 去遍历 `$.ajaxSettings` ，作为配置的默认值。

配置处理完毕后，调用 `ajaxStart` 函数，触发 `ajaxStart` 事件。

#### 判断是否跨域

```javascript
originAnchor = document.createElement('a')
originAnchor.href = window.location.href

if (!settings.crossDomain) {
  urlAnchor = document.createElement('a')
  urlAnchor.href = settings.url
  // cleans up URL for .href (IE only), see https://github.com/madrobby/zepto/pull/1049
  urlAnchor.href = urlAnchor.href
  settings.crossDomain = (originAnchor.protocol + '//' + originAnchor.host) !== (urlAnchor.protocol + '//' + urlAnchor.host)
}
```

如果跨域 `crossDomain` 没有设置，则需要检测请求的地址是否跨域。

`originAnchor` 是当前页面链接，整体思路是创建一个 `a` 节点，将 `href` 属性设置为当前请求的地址，然后获取节点的 `protocol` 和 `host`，看跟当前页面的链接用同样方式拼接出来的地址是否一致。

注意到这里的 `urlAnchor` 进行了两次赋值，这是因为 `ie` 默认不会对链接 `a` 添加端口号，但是会对 `window.location.href` 添加端口号，如果端口号为 `80` 时，会出现不一致的情况。具体见:[pr#1049](https://github.com/madrobby/zepto/pull/1049)

#### 处理请求地址

```javascript
if (!settings.url) settings.url = window.location.toString()
if ((hashIndex = settings.url.indexOf('#')) > -1) settings.url = settings.url.slice(0, hashIndex)
serializeData(settings)
```

如果没有配置 `url` ，则用当前页面的地址作为请求地址。

如果请求的地址带有 `hash`， 则将 `hash` 去掉，因为 `hash` 并不会传递给后端。

然后调用 `serializeData` 方法来序列化请求参数 `data`。

#### 处理缓存

```javascript
var dataType = settings.dataType, hasPlaceholder = /\?.+=\?/.test(settings.url)
if (hasPlaceholder) dataType = 'jsonp'

if (settings.cache === false || (
  (!options || options.cache !== true) &&
  ('script' == dataType || 'jsonp' == dataType)
))
  settings.url = appendQuery(settings.url, '_=' + Date.now())
```

`hasPlaceholder` 的正则匹配规则跟上面分析到 `jsonp` 的替换 `callbackName` 的正则一样，约定以这样的方式来替换 `url` 中的 `callbackName`。因此，也可以用这样的正则来判断是否为 `jsonp`。

如果 `cache` 的配置为 `false` ，或者在 `dataType` 为 `script` 或者 `jsonp` 的情况下， `cache` 没有设置为 `true` 时，表示不需要缓存，清除浏览器缓存的方式也很简单，就是往请求地址的后面加上一个时间戳，这样每次请求的地址都不一样，浏览器自然就没有缓存了。

#### 处理jsonp

```javascript
if ('jsonp' == dataType) {
  if (!hasPlaceholder)
    settings.url = appendQuery(settings.url,
                               settings.jsonp ? (settings.jsonp + '=?') : settings.jsonp === false ? '' : 'callback=?')
  return $.ajaxJSONP(settings, deferred)
}
```

判断 `dataType` 的类型为 `jsonp` 时，会对 `url` 进行一些处理。

如果还没有 `?=` 占位符，则向 `url` 中追加占位符。

如果 `settings.jsonp` 存在，则追加 `settings.jsonp`  + `=?`。

如果 `settings.jsonp` 为 `false`， 则不向 `url` 中追加东西。

否则默认追加 `callback=?`。

`url` 拼接完毕后，调用 `$.ajaxJSONP` 方法，发送 `jsonp` 请求。

#### 一些变量

```javascript
var mime = settings.accepts[dataType],
    headers = { },
    setHeader = function(name, value) { headers[name.toLowerCase()] = [name, value] },
    protocol = /^([\w-]+:)\/\//.test(settings.url) ? RegExp.$1 : window.location.protocol,
    xhr = settings.xhr(),
    nativeSetHeader = xhr.setRequestHeader,
    abortTimeout

if (deferred) deferred.promise(xhr)
```

`mime` 获取数据的 `mime` 类型。

`headers` 为请求头。

`setHeader` 为设置请求头的方法，其实是往 `headers` 上增加对应的 `key` `value` 值。

`protocol` 为协议，匹配一个或多个以字母、数字或者 `-` 开头，并且后面为 `://` 的字符串。优先从配置的 `url` 中获取，如果没有配置 `url`，则取 `window.location.protocol`。

`xhr` 为 `XMLHttpRequest` 实例。

`nativeSetHeader` 为 `xhr` 实例上的 `setRequestHeader` 方法。

`abortTimeout` 为超时定时器的 `id`。

如果 `deferred` 对象存在，则调用 `promise` 方法，以 `xhr` 为基础生成一个 `promise` 。

#### 设置请求头

```javascript
if (!settings.crossDomain) setHeader('X-Requested-With', 'XMLHttpRequest')
setHeader('Accept', mime || '*/*')
if (mime = settings.mimeType || mime) {
  if (mime.indexOf(',') > -1) mime = mime.split(',', 2)[0]
  xhr.overrideMimeType && xhr.overrideMimeType(mime)
}
if (settings.contentType || (settings.contentType !== false && settings.data && settings.type.toUpperCase() != 'GET'))
  setHeader('Content-Type', settings.contentType || 'application/x-www-form-urlencoded')

if (settings.headers) for (name in settings.headers) setHeader(name, settings.headers[name])
xhr.setRequestHeader = setHeader
```

如果不是跨域请求时，设置请求头 `X-Requested-With` 的值为 `XMLHttpRequest` 。这个请求头的作用是告诉服务端，这个请求为 `ajax` 请求。

 `setHeader('Accept', mime || '*/*')` 用来设置客户端接受的资源类型。

当 `mime` 存在时，调用 `overrideMimeType` 方法来重写 `response` 的 `content-type` ，使得服务端返回的类型跟客户端要求的类型不一致时，可以按照指定的格式来解释。具体可以参见这篇文章 《[你真的会使用XMLHttpRequest吗？](https://segmentfault.com/a/1190000004322487)》。

如果有指定 `contentType` ，

或者 `contentType` 没有设置为 `false` ，并且 `data` 存在以及请求类型不为 `GET` 时，设置 `Content-Type` 为指定的 `contentType` ，在没有指定时，设置为 `application/x-www-form-urlencoded` 。所以没有指定 `contentType` 时， `POST` 请求，默认的 `Content-Type` 为 `application/x-www-form-urlencoded`。

如果有配置 `headers` ，则遍历 `headers` 配置，分别调用 `setHeader` 方法配置。

#### before send

```javascript
if (ajaxBeforeSend(xhr, settings) === false) {
  xhr.abort()
  ajaxError(null, 'abort', xhr, settings, deferred)
  return xhr
}
```

调用 `ajaxBeforeSend` 方法，如果返回的为 `false` ，则中止 `ajax` 请求。

#### 同步和异步请求的处理

```javascript
var async = 'async' in settings ? settings.async : true
xhr.open(settings.type, settings.url, async, settings.username, settings.password)
```

如果有配置 `async` ，则采用配置中的值，否则，默认发送的是异步请求。

接着调用 `open` 方法，创建一个请求。

#### 创建请求后的配置

```javascript
if (settings.xhrFields) for (name in settings.xhrFields) xhr[name] = settings.xhrFields[name]

for (name in headers) nativeSetHeader.apply(xhr, headers[name])
```

 如果有配置 `xhrFields` ，则遍历，设置对应的 `xhr` 属性。

再遍历上面配置的 `headers` 对象，调用 `setRequestHeader` 方法，设置请求头，注意这里的请求头必须要在 `open` 之后，在 `send` 之前设置。

#### 发送请求

```javascript
xhr.send(settings.data ? settings.data : null)
```

发送请求很简单，调用 `xhr.send` 方法，将配置中的数据传入即可。

#### 请求响应成功后的处理

```javascript
xhr.onreadystatechange = function(){
  if (xhr.readyState == 4) {
    xhr.onreadystatechange = empty
    clearTimeout(abortTimeout)
    var result, error = false
    if ((xhr.status >= 200 && xhr.status < 300) || xhr.status == 304 || (xhr.status == 0 && protocol == 'file:')) {
      dataType = dataType || mimeToDataType(settings.mimeType || xhr.getResponseHeader('content-type'))

      if (xhr.responseType == 'arraybuffer' || xhr.responseType == 'blob')
        result = xhr.response
      else {
        result = xhr.responseText

        try {
          // http://perfectionkills.com/global-eval-what-are-the-options/
          // sanitize response accordingly if data filter callback provided
          result = ajaxDataFilter(result, dataType, settings)
          if (dataType == 'script')    (1,eval)(result)
          else if (dataType == 'xml')  result = xhr.responseXML
          else if (dataType == 'json') result = blankRE.test(result) ? null : $.parseJSON(result)
        } catch (e) { error = e }

        if (error) return ajaxError(error, 'parsererror', xhr, settings, deferred)
      }

      ajaxSuccess(result, xhr, settings, deferred)
    } else {
      ajaxError(xhr.statusText || null, xhr.status ? 'error' : 'abort', xhr, settings, deferred)
    }
  }
}
```

##### readyState

`readyState` 有以下5种状态，状态切换时，会响应 `onreadystatechange` 的回调。

| 0    | `xhr` 实例已经创建，但是还没有调用 `open` 方法。 |
| ---- | ------------------------------- |
| 1    | 已经调用 `open` 方法                  |
| 2    | 请求已经发送，可以获取响应头和状态 `status`      |
| 3    | 下载中，部分响应数据已经可以使用                |
| 4    | 请求完成                            |

具体见 [MDN:XMLHttpRequest.readyState](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/readyState)

##### 清理工作

```javascript
xhr.onreadystatechange = empty
clearTimeout(abortTimeout)
```

当 `readyState` 变为 `4` 时，表示请求完成（无论成功还是失败），这时需要将 `onreadystatechange` 重新赋值为 `empty` 函数，清除超时响应定时器，避免定时器超时的任务执行。

##### 成功状态判断

```javascript
if ((xhr.status >= 200 && xhr.status < 300) || xhr.status == 304 || (xhr.status == 0 && protocol == 'file:')) {
          ...
   }
```

这里判断的是 `http` 状态码，状态码的含义可以参考 [HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)。

解释一下最后这个条件 `xhr.status == 0 && protocol == 'file:'`。

`status` 为 `0` 时，表示请求并没有到达服务器，有几种情况会造成 `status` 为 `0` 的情况，例如网络不通，不合法的跨域请求，防火墙拦截等。

直接用本地文件的方式打开，也会出现  `status` 为 `0` 的情况，但是我在 `chrome` 上测试，在这种情况下只能取到 		`status` ， `responseType` 和 `responseText` 都取不到，不清楚这个用本地文件打开时，进入成功判断的目的何在。

##### 处理数据

```javascript
blankRE = /^\s*$/,

dataType = dataType || mimeToDataType(settings.mimeType || xhr.getResponseHeader('content-type'))
if (xhr.responseType == 'arraybuffer' || xhr.responseType == 'blob')
  result = xhr.response
else {
  result = xhr.responseText

  try {
    // http://perfectionkills.com/global-eval-what-are-the-options/
    // sanitize response accordingly if data filter callback provided
    result = ajaxDataFilter(result, dataType, settings)
    if (dataType == 'script')    (1,eval)(result)
    else if (dataType == 'xml')  result = xhr.responseXML
    else if (dataType == 'json') result = blankRE.test(result) ? null : $.parseJSON(result)
  } catch (e) { error = e }
  if (error) return ajaxError(error, 'parsererror', xhr, settings, deferred)
```

首先获取 `dataType`，后面会根据 `dataType` 来判断获得的数据类型，进而调用不同的方法来处理。

如果数据为 `arraybuffer` 或 `blob` 对象时，即为二进制数据时，`result` 从 `response` 中直接取得。

否则，用 `responseText` 获取数据，然后再对数据尝试解释。

在解释数据前，调用 `ajaxDataFilter` 对数据进行过滤。

如果数据类型为 `script` ，则使用 `eval` 方法，执行返回的 `script` 内容。

这里为什么用 `(1, eval)` ，而不是直接用 `eval` 呢，是为了确保 `eval` 执行的作用域是在 `window` 下。具体参考：[(1,eval)('this') vs eval('this') in JavaScript?](https://stackoverflow.com/questions/9107240/1-evalthis-vs-evalthis-in-javascript) 和 《[Global eval. What are the options?](http://perfectionkills.com/global-eval-what-are-the-options/)》

如果 `dataType` 为 `xml` ，则调用`responseXML` 方法

如果为 `json` ，返回的内容为空时，结果返回 `null` ，如果不为空，调用 `$.parseJSON` 方法，格式化为 `json` 格式。相关分析见《[读zepto源码之工具函数](https://github.com/yeyuqiudeng/reading-zepto/blob/4143f028beff94ce3834e41620fdea48b764301c/src/%E8%AF%BBZepto%E6%BA%90%E7%A0%81%E4%B9%8B%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md#parsejson)》

如果解释出错了，则调用 `ajaxError` 方法，触发 `ajaxError` 事件，事件类型为 `parseerror`。

如果都成功了，则调用  `ajaxSuccess` 方法，执行成功回调。

##### 响应出错

```javascript
ajaxError(xhr.statusText || null, xhr.status ? 'error' : 'abort', xhr, settings, deferred)
```

如果 `status` 不在成功的范围内，则调用 `ajaxError` 方法，触发 `ajaxError` 事件。

#### 响应超时

```javascript
if (settings.timeout > 0) abortTimeout = setTimeout(function(){
  xhr.onreadystatechange = empty
  xhr.abort()
  ajaxError(null, 'timeout', xhr, settings, deferred)
}, settings.timeout)
```

如果有设置超时时间，则设置一个定时器，超时时，首先要将 `onreadystatechange` 的回调设置为空函数 `empty` ，避免超时响应执行完毕后，请求完成，再次执行成功回调。

然后调用 `xhr.abort` 方法，取消请求的发送，并且调用 `ajaxError` 方法，触发 `ajaxError` 事件。

### $.get

```javascript
$.get = function(/* url, data, success, dataType */){
  return $.ajax(parseArguments.apply(null, arguments))
}
```

`$.get` 是 `$.ajax` `GET` 请求的便捷方法，内部调用了 `$.ajax` ，不需要指定请求类型。

### $.post

```javascript
$.post = function(/* url, data, success, dataType */){
  var options = parseArguments.apply(null, arguments)
  options.type = 'POST'
  return $.ajax(options)
}

```

 `$.post` 是 `$.ajax` `POST` 请求的便捷方法，跟 `$.get` 一样，只开放了 `url`、`data` 、`success` 和 `dataType` 等几个接口参数，默认配置了 `type` 为 `POST` 请求。	

### $.getJSON

```javascript
$.getJSON = function(/* url, data, success */){
  var options = parseArguments.apply(null, arguments)
  options.dataType = 'json'
  return $.ajax(options)
}
```

`$.getJSON` 跟 `$.get` 差不多，比 `$.get` 更省了一个 `dataType` 的参数，这里指定了 `dataType` 为 `json` 类型。

### $.fn.load

```javascript
$.fn.load = function(url, data, success){
  if (!this.length) return this
  var self = this, parts = url.split(/\s/), selector,
      options = parseArguments(url, data, success),
      callback = options.success
  if (parts.length > 1) options.url = parts[0], selector = parts[1]
  options.success = function(response){
    self.html(selector ?
              $('<div>').html(response.replace(rscript, "")).find(selector)
              : response)
    callback && callback.apply(self, arguments)
  }
  $.ajax(options)
  return this
}
```

 `load` 方法是用 `ajax` 的方式，请求一个 `html` 文件，并将请求的文件插入到页面中。

`url` 可以指定选择符，选择符用空格分割，如果有指定选择符，则只将匹配选择符的文档插入到页面中。`url` 的格式为 `请求地址 选择符`。

 ```javascript
var self = this, parts = url.split(/\s/), selector,
    options = parseArguments(url, data, success),
    callback = options.success
if (parts.length > 1) options.url = parts[0], selector = parts[1]
 ```

`parts` 是用空格分割后的结果，如果有选择符，则 `length` 会大于 `1`，数组的第一项为请求地址，第二项为选择符。

调用 `parseArguments` 用来重新调整参数，因为 `data` 和 `success` 都是可选的。

```javascript
options.success = function(response){
  self.html(selector ?
            $('<div>').html(response.replace(rscript, "")).find(selector)
            : response)
  callback && callback.apply(self, arguments)
}
```

请求成功后，如果有 `selector` ，则从文档中筛选符合的文档插入页面，否则，将返回的文档全部插入页面。

如果有配置回调函数，则执行回调。

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
* [一个普通的 Zepto 源码分析（二） - ajax 模块](http://www.cnblogs.com/BlackStorm/p/Zepto-Analysing-For-Ajax-Module.html)
* [MDN:XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest)
* [fetch.spec.whatwg.org](https://fetch.spec.whatwg.org/#responses)
* [HTTP status code 0 - what does this mean for fetch, or XMLHttpRequest?](https://stackoverflow.com/questions/872206/http-status-code-0-what-does-this-mean-for-fetch-or-xmlhttprequest)
* [(1,eval)('this') vs eval('this') in JavaScript?](https://stackoverflow.com/questions/9107240/1-evalthis-vs-evalthis-in-javascript)
* [Global eval. What are the options?](http://perfectionkills.com/global-eval-what-are-the-options/)

## License

[署名-非商业性使用-禁止演绎 4.0 国际 (CC BY-NC-ND 4.0)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

最后，所有文章都会同步发送到微信公众号上，欢迎关注,欢迎提意见：  ![](https://raw.githubusercontent.com/yeyuqiudeng/resource/master/images/qrcode_front-end-article.jpg) 

作者：对角另一面

