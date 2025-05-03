+++
date = '2025-05-04T00:19:00+09:00'
draft = false
title = 'PWA 应用图标：如何兼顾 macOS、Windows、iPad'
categories= ['Programming']
tags= ['PWA']
+++

首先看 macOS 任务栏的图标规律：
![icon-list](/img/icon-list.webp)

会发现第三个：YouTube Music（PWA 版本）的图标很不对劲，不是因为它是圆形，其他都是方形。而是它很大，宽高明显大过其他图标。在我对比了 VSC 和 YouTube Music 的图标图片之后发现了区别所在：
![youtube-music](/img/youtube-music.webp)
![vscode](/img/vscode.webp)

上面看不明显的话，这么对比看就明显了：
![youtube-music-vscode](/img/youtube-music-vscode.webp)

> 解释下，图 3 是图 2 在控制台（F12）资源那里显示的图片，之所以看这个是因为可以看出来，图 2 外层有一圈白色的，其实也是图片的一部分，而且是透明的。
> 图 1 四角也有白色的，也是透明的。

所以现在能解释为什么 VSC 和 YouTube Music 的图标就算尺寸一样大，他们在 macOS 上显示的大小也不同了：VSC 有一圈透明白边，YM 没有。

在做 PWA 项目时，会涉及到给 PWA 添加桌面图标的情况，即，在 [`manifest.icons`](https://developer.mozilla.org/zh-CN/docs/Web/Manifest) 配置中添加图标设置，比如：

```json
{
  "icons": [
    {
      "src": "logo64.png",
      "sizes": "64x64 48x48 32x32 24x24 16x16",
      "type": "image/x-icon"
    },
    {
      "src": "logo192.png",
      "type": "image/png",
      "sizes": "192x192"
    },
    {
      "src": "logo512.png",
      "type": "image/png",
      "sizes": "512x512"
    }
  ]
}
```

在我这边的项目中，上面三个图标文件都是像 VSC 那种有一圈透明边的文件，但我发现到了 Windows 平台，任务栏的图标显示很小（最后一个）：
![windows-icon](/img/windows-icon.webp)
和左边 Brave 浏览器对比下，显得更小。一开始我以为 Windows PWA 图标取的是上面的`icons`，所以我把`logo64.png`、`logo192.png`、`logo512.png`的图标都去掉了透明白边，发现到了 Windows 这边还是不行。

我最后才发现：**原来 Windows（只测试了 win7） PWA 应用在任务栏（包括桌面上）的图标取的是网页的`favicon.ico`，而 macOS 平台取的是`manifest.icons`里的图标，真是坑。**

所以**解决方法**就是：在`manifest.icons`配置时用有一圈透明边缘的图（macOS 用），然后 html `favicon.ico`的图用不带透明边缘的图（Windows 用）。

**更新：** Windows11 的 Chrome/Edge 取的是`manifest.icons`配置，这就很尴尬了。如果你配置的图标不留外层一圈透明空白，你的图标在 Windows11 上显示正常，在 macOS 上显示就和上面的 YouTube Music 一样显得大。反之 macOS 上正常，那么 Windows11 上就显得比其他图标小。

然后**iPad**也需要单独处理：

- 首先 Safari 不会读 PWA 的 manifest 配置文件，会去读 html 的`<link rel="apple-touch-icon" href="/apple-pwa.png">`
- 其次，不能有透明像素，透明像素就会显示为黑色。（上面 VSCode 图标最外层那一圈就是透明像素）
- 还有个特点 就是得贴边 外框是 safari 给加（这条不确定，应该是指不应该给图标增加 box-shadow 阴影效果）
- safari 的 title 透明是这样搞：apple-mobile-web-app-capable+apple-mobile-web-app-status-bar-style
  > 然后还有个安全区（刘海，竖屏、横屏）的坑。上面这个设置好之后，safari 就会遵守 theme-color 设置的颜色
