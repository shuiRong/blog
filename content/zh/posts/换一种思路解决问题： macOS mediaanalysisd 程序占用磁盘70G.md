+++
date = '2025-06-09T00:29:39+09:00'
draft = false
title = '换一种思路解决问题： macOS mediaanalysisd 程序占用磁盘70G'
categories= ['Programming']
tags= ['macOS']
+++

最近系统提示我剩余磁盘空间不足，我通过 OmniDiskSweeper 检查后，发现 `/Users/<user_name>/Library/Containers/com.apple.mediaanalysisd/Data/Library/Caches/com.apple.mediaanalysisd` 目录占用了差不多 70G 的大小。
![macos-mediaanalysisd.webp](/img/macos-mediaanalysisd.webp)

### mediaanalysisd 是什么？

它是一个获取照片的面部识别信息、元数据的程序，而且可能还和 macOS 上的 Apple Intelligence 服务有关。它会在背后默默运行，分析电脑上的媒体资源。

不管怎么样，70G 也太过分了。

**我开始尝试各种方法：**

1. 我删掉了这个文件夹，但重启电脑后，系统又重新创建了相同的文件夹（当然，70G 的占用已经没有了，但假以时日，它可能还会增长到几十 G 的大小）
2. 我 kill 掉了进程，但很快它又重新启动。
3. 我尝试删除掉这个 plist 文件，它在：`/System/Library/LaunchAgents/com.apple.mediaanalysisd.plist` ，但系统提示无法删除（因为是操作系统的 plist 文件）。尽管可能可以关闭系统 SIP 后再删除它，但有些麻烦。

非常棘手，最后我思路一转：**不删除它也没关系，只要能保证它不要占用磁盘太多空间即可。**

macOS 无法限制某个文件夹的最大容量，但磁盘映像文件的话，可以限制它的大小。

**所以思路就是这样：**

1. 创建一个磁盘，创建的时候限制该磁盘的大小为 100M
2. 挂载改磁盘
3. 将 `/Users/<user_name>/Library/Containers/com.apple.mediaanalysisd/Data/Library/Caches/com.apple.mediaanalysisd` 目录映射到这个磁盘上。

这就能实现让 mediaanalysisd 进程不要占用我太多磁盘空间的目的了。

具体的命令如下：

> （记得修改 `<user_name>` 为你真实名称）

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

> 只要思想不滑坡，方法总比困难多
