# 附：怎样处理 Safari 移动端对图片资源的限制

本文翻译自《[How to work around the Mobile Safari image resource limit](https://www.fngtps.com/2010/mobile-safari-image-resource-limit-workaround/)》，原文写于2010年10月25日。可能部分限制已经不再适用。

受限于 `Ipad` 和 `Iphone` 的可用内存，`Safari` 浏览器的移动端会比桌面端有着更严格的[资源使用限制](https://developer.apple.com/library/safari/documentation/AppleApplications/Reference/SafariWebContent/CreatingContentforSafarioniPhone/CreatingContentforSafarioniPhone.html#//apple_ref/doc/uid/TP40006482-SW15)

其中之一是每个 `HTML` 页面的图片数据总量。当移动端的 `Safari` 浏览器加载了 `8` 到 `10MB` 的图片数据后，就会停止加载其他图片，甚至浏览器还会崩溃。

 大多数网站都不会受到这条限制的影响，因为保持页面合理的大小通常是一种很聪明的做法。

但是，在下面的场景中，你可能会遇到麻烦，如大型的图片画廊和幻灯片，或者是异步加载新数据的 `web` 应用，例如模拟不同版块切换时的原生动画（是的，你可以用移动端 `Safari` 模拟 `Flipboard` 的切换效果 ）。

<img src="https://www.fngtps.com/2010/mobile-safari-image-resource-limit-workaround/ipad_example1.jpg" width="340px" /><img src="https://www.fngtps.com/2010/mobile-safari-image-resource-limit-workaround/ipad_example2.jpg" width="340px" />

我们有充足的理由相信，只通过删除不再需要的图片元素，就可以不受这条限制的影响：

```javascript
var img = document.getElementById('previous');
img.parentNode.removeChild(img);
```

但是然并卵，因为某些原因，将图片从 `DOM` （或者一个包含图片的元素）中删除时，图片的真实数据并没有释放。真是头大啊！

而将图片的 `src` 属性设置为其他的（更小的）图片链接，却起到了作用。

```javascript
var img = document.getElementById('previous');
img.src = 'images/empty.gif';
```

替换掉 `src` 属性后，旧的图片数据最终得到了释放。

我已经彻底测试过这种方法，下面几个方面是需要注意的：

1. 将 `src` 属性设置为其他图片后，图片数据不会立即释放，需要一段时间让垃圾回收器来真正地释放内存。这意味着，如果你太块地插入图片，依旧可能会陷入麻烦中。
2. 在移动端 `Safari` 触发限制后，即便删除一部分或者全部已经加载的数据，`Safari` 也不会再加载额外的图片，这种情况即便在切换到其他页面时也继续存在。这意味着在测试这项技术时，你需要经常重启 `Safari`（这差点把我逼疯了）。
3. 如果你想将图片元素从 `DOM` 中删除，你还必须确保在更改 `src` 前，元素不能为垃圾回收掉，否则，旧图片数据不会被释放。下面这个是最好的解决方案：

```javascript
var img = document.getElementById('previous');
img.parentNode.removeChild(img);
img.src = 'data:image/gif;base64,' + 
      'R0lGODlhAQABAAD/ACwAAAAAAQABAAACADs=';
window.timeout(function() {
img = null;
}, 60000);
```

你可以看到，我使用了 `data URI` 作为替换图片。

<img src="https://www.fngtps.com/2010/mobile-safari-image-resource-limit-workaround/ipad_without.jpg" width="340px" /><img src="https://www.fngtps.com/2010/mobile-safari-image-resource-limit-workaround/ipad_with.jpg" width="340px" />

（如果你只是删除图片元素， `iPad` 在加载8张图片后会停止继续加载，如果用 `Zepto` 的 `assets` 插件，会持续加载。）

在上周我和 `Thomas Fuchs` 解释了这项技术后，他立即将它加入了 `Zepto` 中。这个周末，我贡献了一个测试函数，你可以自己用它来测试下。

