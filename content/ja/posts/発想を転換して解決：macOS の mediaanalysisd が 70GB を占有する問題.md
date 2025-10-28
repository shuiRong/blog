+++
date = '2025-06-09T00:29:39+09:00'
draft = false
title = '発想を転換して解決：macOS の mediaanalysisd が 70GB を占有する問題'
categories= ['Programming']
tags= ['macOS']
+++

最近、ストレージ残量が少ないと macOS に警告されました。OmniDiskSweeper で調べてみると、`/Users/<user_name>/Library/Containers/com.apple.mediaanalysisd/Data/Library/Caches/com.apple.mediaanalysisd` が約 70GB も占めていました。
![macos-mediaanalysisd.png](/img/macos-mediaanalysisd.png)

### mediaanalysisd とは？

写真から顔情報やメタデータを取得するプロセスで、macOS の Apple Intelligence とも関連している可能性があります。バックグラウンドで動き続け、Mac 上のメディアを解析します。

とはいえ 70GB はさすがにやり過ぎです。

**試したこと：**

1. フォルダを削除してみたところ、再起動後に同じフォルダが再生成されました（占有は一旦解消されましたが、放っておくとまた肥大化しそうです）。
2. プロセスを kill してもすぐに復活します。
3. `/System/Library/LaunchAgents/com.apple.mediaanalysisd.plist` を削除しようとしましたが、システムの保護対象なのでそのままでは削除できません（SIP を無効にすれば可能かもしれませんが手間です）。

どれも決め手に欠けます。そこで発想を変えて、**消せないなら容量を使わせなければいい** と考えました。

macOS では特定フォルダの容量制限はできませんが、ディスクイメージなら上限を設定できます。

**方針は次の通りです。**

1. 上限 100MB のディスクイメージを作成する
2. ディスクイメージをマウントする
3. `/Users/<user_name>/Library/Containers/com.apple.mediaanalysisd/Data/Library/Caches/com.apple.mediaanalysisd` をそのディスクにシンボリックリンクする

これで mediaanalysisd がディスクを肥大化させなくなります。

具体的なコマンドは以下の通りです（`<user_name>` は自分のユーザー名に置き換えてください）。

```bash
⇒  hdiutil create -size 100m -fs APFS -type SPARSEBUNDLE -volname com.apple.mediaanalysisd ~/LimitedFolder100.dmg
created: /Users/<user_name>/LimitedFolder100.dmg.sparsebundle

⇒  hdiutil attach ~/LimitedFolder100.dmg.sparsebundle
/dev/disk4          	GUID_partition_scheme
/dev/disk4s1        	Apple_APFS
/dev/disk5          	EF57347C-0000-11AA-AA11-0030654
/dev/disk5s1        	41504653-0000-11AA-AA11-0030654	/Volumes/com.apple.mediaanalysisd

⇒ ln -s /Volumes/com.apple.mediaanalysisd /Users/<user_name>/Library/Containers/com.apple.mediaanalysisd/Data/Library/Caches
```

> 発想を転換すれば道は開ける。
