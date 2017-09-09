---

---

# 附：怎样处理 Safari 移动端对图片资源的限制

受限于 `Ipad` 和 `Iphone` 的可用内存，`Safari` 浏览器的移动端会比桌面端有着更严格的[资源使用限制](https://developer.apple.com/library/safari/documentation/AppleApplications/Reference/SafariWebContent/CreatingContentforSafarioniPhone/CreatingContentforSafarioniPhone.html#//apple_ref/doc/uid/TP40006482-SW15)

其中一条限制是每个 `HTML` 页面的图片数据总量。当移动端的 `Safari` 浏览器加载了 `8` 到 `10MB` 的图片数据后，就会停止加载其他图片，甚至浏览器还会崩溃。

 大多数网站都不会受到这条限制的影响，因为保持页面合理的大小通常是一种很聪明的做法。

但是，在下面的场景中，你可能会遇到麻烦，如大型的图片画廊和幻灯片，或者是异步加载新数据的 `web` 应用，例如模拟不同版块切换时的原生动画（是的，你可以用移动端 `Safari` 模拟 `Flipboard` 的切换效果 ）。

<img src="https://www.fngtps.com/2010/mobile-safari-image-resource-limit-workaround/ipad_example1.jpg" width="340px" /><img src="https://www.fngtps.com/2010/mobile-safari-image-resource-limit-workaround/ipad_example2.jpg" width="340px" />

