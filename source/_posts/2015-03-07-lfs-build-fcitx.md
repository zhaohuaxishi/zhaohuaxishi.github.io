---
author: 郭荣飞
date: 2015-03-07
title: LFS 中编译 Fcitx
tags:
 - LFS
---


`LFS` 教会我从无到有构建一个 `Linux` 系统，也让我对 `Linux` 系统的内部工作原理
有了更加深刻的理解。但是 `LFS` 和 `BLFS` 终究还是外国人写的，所以在中文支持方面
并不是特别让人满意。要想自己编译一个可以供日常使用的 `Linux` 系统对于我来说最大
的障碍在于 `BLFS` 中没有关于如何编译输入法的内容，而且网上能够找到的关于如何编译
输入法的资料几乎为零。

<!--more-->

当然 `BLFS` 早就想到了这个问题，它不可能包含所有可能的包，所以它在开头第二章的
`Going Beyond BLFS` 一节中就提到了一些非常有用的建议，比如去下载你想要编译的软件
的 `Arch` 对应的包，解压之后查看里面的 `PKGBUILD` 文件以便得到如何编译你想要的包
的办法。 我根据这种方式成功的编译了 `Fcitx` 输入法框架以及 `fcitx-googlepinyin`
输入法。

因为网上关于如何编译输入法的资料较少，这里我把我的编译过程写下来希望对大家有帮助，
当然由于个人经验的缺乏，这个编译过程可能存在漏洞，如果你有更好的编译方法请
[联系我](mailto:guorongfei@126.com) 以便我及时更正，以免误人子弟。

本文将会采用 `BLFS` 的编写风格。我觉得这种风格比较适合阅读，如果文中没有特别提及，
那么你应该首先解压下载的源码包，然后切换到解压得到的目录内再执行后续的命令。编译
和安装完成之后，切换到源码包所在的同一级目录，删除解压出来的源码文件夹和编译时
新建的文件夹（如果存在的话）。

本文中提到的两个依赖包 `Presage` 和 `opencc` 并没有包含在 `BLFS` 中，请参考给出
的链接中的相关文档自行编译。

# Fcitx-4.2.8.5#

## fcitx 简介##

`Fcitx` 是一个支持扩展的输入法框架，目前支持 `Linux` 操作系统以及像 `freebsd`
这类的 `Uniux` 系统。它包含三个内置的输入法引擎 `Pinyin`，`QuWei` 和
`基于 Table` 的输入法。`Fcitx` 尽量在所有的桌面环境中都提供原生的体验以及轻量级
的引擎核心。你可以很容易的根据自己的需求来配置它。 这个包可以在 `LFS-7.6` 平台上
正常的编译和使用。

## 源码包信息##

  - 下载地址（HTTP）：<http://download.fcitx-im.org/fcitx/fcitx-4.2.8.5_dict.tar.xz>

  - 下载包的 MD5 sum：8cf81a003a02b2f16d04dfec6e82e90a

  - 下载包的大小：8.3 MB

  - 预计需要的空间大小： 未知

  - 预计需要的编译时间： 1 SBU

## 附加的下载##

  - 需要的补丁：<https://projects.archlinux.org/svntogit/community.git/plain/trunk/custom-translation-install-dir.patch?h=packages/fcitx>

  - 需要的补丁：<https://projects.archlinux.org/svntogit/community.git/plain/trunk/opencc-1.0.patch?h=packages/fcitx>

  - 配置工具：<http://download.fcitx-im.org/fcitx-configtool/fcitx-configtool-0.4.8.tar.xz>

  - 谷歌拼音输入法：

      - 库程序：<https://libgooglepinyin.googlecode.com/files/libgooglepinyin-0.1.2.tar.bz2>
      - `fcitx-googlepinyin`：<http://download.fcitx-im.org/fcitx-googlepinyin/fcitx-googlepinyin-0.1.6.tar.xz>

## Fcixt 的依赖##

### 需要的依赖###

[LLVM-3.5.0][]（需要 Clang），
[CMake-3.0.1][]，
[Xorg Libraries][]，
[GTK+-2.24.24][]，
[Qt-4.8.6][]，
[D-Bus-1.8.8][]，
[dbus-glib-0.102][]，
[ICU-53.1][]，
[Pango-1.36.7][]，
[Cairo-1.12.16][]，
[enchant-1.6.0][]，
[Presage][]，
[gobject-introspection-1.40.0][]，
[libxml2-2.9.1][]，
[ISO Codes-3.56][]

  [LLVM-3.5.0]: http://www.linuxfromscratch.org/blfs/view/7.6/general/llvm.html
  [CMake-3.0.1]: http://www.linuxfromscratch.org/blfs/view/7.6/general/cmake.html
  [Xorg Libraries]: http://www.linuxfromscratch.org/blfs/view/7.6/x/x7lib.html
  [GTK+-2.24.24]: http://www.linuxfromscratch.org/blfs/view/7.6/x/gtk2.html
  [Qt-4.8.6]: http://www.linuxfromscratch.org/blfs/view/7.6/x/qt4.html
  [D-Bus-1.8.8]: http://www.linuxfromscratch.org/blfs/view/7.6/general/dbus.html
  [dbus-glib-0.102]: http://www.linuxfromscratch.org/blfs/view/7.6/general/dbus-glib.html
  [ICU-53.1]: http://www.linuxfromscratch.org/blfs/view/7.6/general/icu.html
  [Pango-1.36.7]: http://www.linuxfromscratch.org/blfs/view/7.6/x/pango.html
  [Cairo-1.12.16]: http://www.linuxfromscratch.org/blfs/view/7.6/x/cairo.html
  [enchant-1.6.0]: http://www.linuxfromscratch.org/blfs/view/7.6/general/enchant.html
  [Presage]: http://presage.sourceforge.net/
  [gobject-introspection-1.40.0]: http://www.linuxfromscratch.org/blfs/view/7.6/general/gobject-introspection.html
  [libxml2-2.9.1]: http://www.linuxfromscratch.org/blfs/view/7.6/general/libxml2.html
  [ISO Codes-3.56]: http://www.linuxfromscratch.org/blfs/view/7.6/general/iso-codes.html

### 可选的依赖###

[opencc][]（简繁体中文转换）和 [GTK+-3.12.2][]

  [opencc]: https://bintray.com/byvoid/opencc/OpenCC
  [GTK+-3.12.2]: http://www.linuxfromscratch.org/blfs/view/7.6/x/gtk3.html

## Fcitx 的安装##

首先打上必要的补丁：

	patch -p1 -i ../custom-translation-install-dir.patch &&
	patch -p1 -i ../opencc-1.0.patch &&
	cd ..

`Fcitx` 文档中推荐建立 `build` 文件夹编译：

	mkdir fcitx-build &&
	cd fcitx-build

使用下面的命令编译 `Fcitx`：

	cmake ../fcitx-4.2.8.5 \
	    -DCMAKE_BUILD_TYPE=Release \
	    -DCMAKE_INSTALL_PREFIX=/usr \
	    -DSYSCONFDIR=/etc \
	    -DFORCE_PRESAGE=ON \
	    -DFORCE_ENCHANT=ON \
	    -DENABLE_TEST=ON \
	    -DENABLE_GTK2_IM_MODULE=ON \
	    -DENABLE_QT_IM_MODULE=ON  &&
	make

编译完成之后通过以下命令测试编译结果： `make test`。

接写来使用超级用户权限执行以下命令以安装 `Fcitx`：

	make install &&
	gtk-update-icon-cache -q -t -f /usr/share/icons/hicolor &&
	update-desktop-database -q &&
	update-mime-database /usr/share/mime &> /dev/null &&

	cd src/frontend/gtk2 &&
	make install &&
	gtk-query-immodules-2.0 --update-cache &&
	cd ../../../ &&

	cd src/frontend/qt &&
	make install &&
	cd ../../../ &&

	cd tools/gui &&
	make install &&
	cd ../../ &&

	cd src/lib/fcitx-qt &&
	make install

如果你下载了 `fcitx-configtool` 可以执行下面的命令编译它：

	cmake -DCMAKE_INSTALL_PREFIX=/usr . &&
	make

编译完成之后以管理员身份执行： `make install` 安装这个包。

如果你下载了额外的谷歌拼音输入法，你可以通过下面的方式编译它。首先在解压的源文件
目录的同级目录下执行下面的命令编译库文件：

	rm -rf build &&
	mkdir build &&
	cd build &&
	cmake -DCMAKE_INSTALL_PREFIX=/usr . \
	      -DENABLE_STATIC=Off ../libgooglepinyin-0.1.2 &&
	make

以超级用户权限编译执行下面这个包以安装这个包： `make install`

然后通过下面的方式编译 `fcitx-googlepinyin` 程序，首先切换到 `fcit-googlepinyin`
解压后得到的目录中：

	rm -rf build &&
	mkdir build &&
	cd build &&
	cmake -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_BUILD_TYPE=Release .. &&
	make

然后以超级用户权限执行以下命令安装：

	make install &&
	gtk-update-icon-cache -q -t -f /usr/share/icons/hicolor

安装的谷歌拼音输入法可以被 `Fcitx` 自动识别，不需要额外的配置。

## 包的内容##

   **安装的程序：** fcitx，fcitx4-config，fcitx-autostart，fcitx-configtool，fcitx-dbus-watcher，fcitx-diagnose，fcitx-remote，fcitx-skin-installer

   **安装的库：** libfcitx\*.so.\*

   **安装的目录：** /usr/include/fcitx，/usr/include/fcitx-config，/usr/include/fcitx-gclient，/usr/include/fcitx-utils

## 命令简介##

   **fcitx**: 用来启动 `Fcitx` 框架

   **fcitx-autostart**: `fcitx` 命令的一个封装

   **fcitx-configtool**： `Fcitx` 的配置程序，如果安装了 `fcitx-configtool` 包，它会调用 `fcitx-config-gtk`。

## 配置##

为了启动 `Fcitx` 框架，你可以执行 `fcitx-autostart` 命令，如果需要开机自启动可以
把这条命令加入到自启动脚本中，比如在 openbox 中可以把它加入到
`./config/openbox/autostart` 文件中。

你可以直接调用 `fcitx-configtool` 命令来配置 `Fcitx` 本身。如果你安装了额外的
配置程序，这条命令会调用相应的配置程序；如果没有的话，它会调用 `EDITOR` 环境变量
指定的编辑器直接打开配置文件直接编辑配置文件以达到配置的目的。

