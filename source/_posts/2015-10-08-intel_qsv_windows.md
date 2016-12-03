---
author: 郭荣飞
date: 2015-10-08 02:00:00
title: 英特尔 QSV 在 FFMPEG 中的使用（Windows）
tags:
 - GPU
---

上一篇文章中介绍了 Linux 下如何在 FFMPEG 中使用 Intel QSV，这篇文章介绍如
何在 Windows 下使用这个功能。

# INDE

在 Windows 下通常使用 `INDE` 中的 `Intel Media SDK` 而不是 `MMS`，因为后者
只在 `Linux` 下和 `Windows Server` 下可用。

[INDE][] 可以免费下载，建议下载它的离线安装包，因为很多功能你并不需要，使
用离线安装包，你可以指下载你想要的功能。

[INDE]: https://software.intel.com/en-us/intel-inde

<!--more-->

# 安装 `Media SDK`

在 Windows 上安装 `Media SDK` 比较简单，请参考[这个链接
][install_media_sdk]中的安装方法。我们只使用它做视频编码，所以只需要选择
`build` 下的

    - Media SDK for Windows
    - Media Raw Acclecerator for Windows

这两项就可以了。

[install_media_sdk]: https://software.intel.com/zh-cn/articles/quick-installation-guide-for-media-sdk-on-windows-with-intel-inde?language=en

# Windows 下编译支持 qsv 的 FFMPEG 库

## 编译 `mfx_dispatcher`

windows 编译 qsv 之前需要安装 `mfx_dispatcher`，它相当于是应用程序和具体的
硬件加速库之间的一个中间层，它负责帮助应用库定位底层代码，这样应用库就可以
不用直接链接到硬件加速的具体实现。

[`mfx_dispatcher`][] 代码可以在 `github` 上下载到，在 github 的 README 中也
提供了编译方法。需要注意的是，它使用的编译工具是 `mingw64` 的 `x86_64` 工
具链，如果你使用的是 `mingw64` 的 `i686` 工具库，记得把教程中的 `x86_64`
替换成 `i686`。

[mfx_dispatcher]: https://github.com/lu-zero/mfx_dispatch

`mfx_dispatcher` 安装完成之后会在 `/usr/i686-w64-mingw32/usr/local/` 下生成
相应的库文件和头文件。

## 链接到 `FFMPEG`

FFMPEG 需要使用 `pkg-config` 定位 `libmfx` 库，这个库的 `libmfx.pc` 文件在
安装完 `mfx_dispatcher` 之后会安装在
`/usr/i686-w64-mingw32/usr/local/lib/pkgconfig` 目录下。为了让 `FFMPEG` 的
`configure` 脚本能够找到它你需要把这个地址加入到 `PKG_CONFIG_PATH` 中。

    export PKG_CONFIG_PATH=/usr/i686-w64-mingw32/usr/local/lib/pkgconfig

为了让 `FFMPEG` 支持 `qsv` 你需要加入下面三个配置选项：

    ./configure --enable-libmfx \
                --enable-encoder=h264_qsv \
                --enable-decoder=h264_qsv \
                ...


# 使用中可能会出现的问题

在使用 `h264_qsv` 编码器的时候，可能会出现 `Error initializing an internal
MFX session` 错误，目前没有找到具体原因。在把 Media SDK 下的
`libmfxhw32.dll` 文件拷贝到执行目录下之后这个问题就消失看。
