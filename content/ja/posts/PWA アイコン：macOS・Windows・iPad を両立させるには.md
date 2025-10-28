+++
date = '2025-05-04T00:19:00+09:00'
draft = false
title = 'PWA アイコン：macOS・Windows・iPad を両立させるには'
categories= ['Programming']
tags= ['PWA']
+++

まずは macOS の Dock（アプリケーションのアイコンが並ぶバー）に表示されるアイコンの傾向を観察してみましょう。
![icon-list](/img/icon-list.webp)

3 つ目の YouTube Music（PWA バージョン）のアイコンに違和感があるのが分かります。丸いからではなく、ほかのアイコンより明らかに大きく表示されているのです。VSC と YouTube Music のアイコンファイルを比較したところ、決定的な違いがありました。
![youtube-music](/img/youtube-music.webp)
![vscode](/img/vscode.webp)

上の画像だけでは分かりづらい場合、この比較を見ると一目瞭然です。
![youtube-music-vscode](/img/youtube-music-vscode.webp)

> 図 3 は、図 2 をブラウザーのコンソール（F12）の「リソース」ビューで表示したものです。図 2 の外側に白い輪があるのが分かりますが、これは画像の一部であり、透明なピクセルになっています。  
> 図 1 の四隅にある白い部分も同様に透明です。

つまり VSC と YouTube Music はファイルのサイズこそ同じですが、macOS 上での表示サイズが異なる理由はここにあります。VSC のアイコンには透明の白フチがあり、YouTube Music にはありません。

PWA プロジェクトでデスクトップアイコンを用意する際には、[`manifest.icons`](https://developer.mozilla.org/zh-CN/docs/Web/Manifest) にアイコンを設定します。たとえば以下のような構成です。

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

私のプロジェクトでは、これら 3 つのアイコンを VSC と同じく透明の余白つきにしていました。しかし Windows のタスクバーで確認すると、アイコンが極端に小さく表示されていました（一番右側）。
![windows-icon](/img/windows-icon.webp)

左側の Brave ブラウザーと比べると小ささが際立ちます。最初は Windows の PWA アイコンも上記 `icons` から参照されると考え、`logo64.png`、`logo192.png`、`logo512.png` から透明の余白を取り除いてみましたが、それでも改善しませんでした。

最終的に分かったのは、**Windows（私が検証したのは Windows 7）の PWA アプリは、タスクバー（およびデスクトップ）で表示するアイコンに `favicon.ico` を使い、macOS は `manifest.icons` の設定を参照する**ということです。落とし穴でした。

したがって**対処法**は、`manifest.icons` には透明の余白付きアイコンを設定して macOS 対応とし、HTML の `favicon.ico` には余白のないアイコンを用意して Windows 対応とする、という二刀流にすることです。

**更新情報：** Windows 11 の Chrome/Edge は `manifest.icons` を参照するようです。困ったことに、外側に余白を入れずに設定すると Windows 11 ではちょうど良いサイズで表示されますが、macOS では YouTube Music のように巨大に見えてしまいます。逆に余白を付ければ macOS では理想的なサイズになるものの、Windows 11 では他のアイコンより小さくなります。

さらに **iPad** も別途対応が必要です。

- Safari は PWA の manifest を読み取らず、HTML の `<link rel="apple-touch-icon" href="/apple-pwa.png">` を参照する
- 透明ピクセルは使用できない（透明部分は黒く表示される。上記 VSCode の外周が透明ピクセルにあたる）
- アイコンは外枠ぴったりに配置する必要があり、外枠自体は Safari が付け加える（box-shadow などの余計な装飾は付けない方が良さそう）
- Safari のタイトルバーを透明にするには `apple-mobile-web-app-capable` と `apple-mobile-web-app-status-bar-style` を併用する  
  > さらにノッチ付きデバイス（縦横どちらの向きでも）ではセーフエリアへの配慮が必要です。上記の設定を済ませると、Safari は `theme-color` で指定した色を尊重するようになります。
