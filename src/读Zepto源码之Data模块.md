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