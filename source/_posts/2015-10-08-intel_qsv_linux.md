---
author: 郭荣飞
date: 2015-10-08 01:00:00
title: 英特尔 QSV 在 FFMPEG 中的使用（Linux）
tags:
 - GPU
---

# Intel Media SDK

现在 `Intel` 不再发布单独的 `Intel Media SDK`， 这个组件在 Linux 平台下集
成在 `Intel Media Server Studio` 中，后文简称 `MMS`

目前的 `MMS` 版本推荐的安装平台只有一个 `CentOS`（`SUSE12` 在当前版本中也
算是一个推荐平台）。其他平台的安装比较复杂，官方也不太推荐使用。后文的介绍
是基于 `CentOS` 操作系统的。

<!--more-->

# 如何安装 MMS

首先，你需要在 [Intel Developer Zone][dev_zone] 下载最新的 `MMS` 版本，其
中的 `Community` 版本是免费的， `MMS` 的安装主要分三个步骤。

[dev_zone]: https://software.intel.com/en-us/intel-media-server-studio

在解压出来的的文件夹下面有一个 `SDK2015Production*` 目录，切换到这个目录下
面之后，有一个 `CentOS` 目录。这个目录下面有一个 `intel_scripts_centos*`
压缩包，解压这个压缩包之后可以得到下面三个脚本：

    -build_kernel_rpm_CentOS.sh
    -install_sdk_UMD_CentOS.sh
    -uninstall_sdk_UMD_CentOS.sh

安装需要用的是前面两个脚本。

## 1. 安卓用户空间驱动（user-mode driver -- UMD）

下面的命令需要使用超级用户权限：

    ./install_sdk_UMD_CentOS.sh

    mkdir /MSS

    chown {普通用户名}:{普通组名} /MSS

## 2. 编译内核空间的驱动包

下面的命令使用普通用户权限执行：

    cp build_kernel_rpm_CentOS.sh /MSS

    cd /MSS

    ./build_kernel_rpm*.sh

## 3. 安装内核空间的驱动

下面的命令使用超级用户权限执行：

    cd /MSS/rpmbuild/RPMS/x86_64

    rpm -Uvh kernel-3.10.*.rpm

    reboot

# 判断是否已经成功的编译内核模块驱动

重启系统之后执行如下命令：

    lsmod | grep 'i915'

得到的类似如下的结果：

    i915                837369 4
    drm_kms_helper      44256 1 i915
    drm                 294746 3 i915,drm_kms_helper
    i2c_algo_bit        13509 1 i915
    intel_gtt           19747 1 i915
    i2c_core            40683 5
    i2c_i801,i915,drm_kms_helper,drm,i2c_algo_bit
    video               19785 1 i915
    button              13953 1 i915

# 如何在 FFMPEG 中编译 intel qsv 硬件编码器

`FFMPEG` 中使用 `libmfx` 实现 `intel qsv` 的硬件编码器，如果想要编译它的硬
件编码器，所以如果想要编译这个硬件编码器，你需要在加入如下的配置选项：

    ./configure --enable-libmfx \
                --enable-encoder=h264_qsv \
                --enable-decoder=h264_qsv \
                ...

## `libmfx can not found using pkg-config`

### `libmfx.pc`

编译中可能会报出下面的错误： `libmfx can not found using pkg-config`，这个
错误可能是不同的原因导致，你需要查看 ffmpeg 根目录下的 `config.log` 文件。

如果这个文件中报错说 pkg-config 无法找到 libmfx 这个库，那是因为 `MMS` 的
默认安装没有提供 `libmfx.pc` 文件，你需要在自己创建这个文件：

    sudo mkdir -p /opt/intel/mediasdk/pkgconfig

    vim /opt/intel/mediasdk/lib64/pkgconfig/libmfx.pc

在文件中写入如下内容：

    prefix=/opt/intel/mediasdk
    exec_prefix=${prefix}
    libdir=${exec_prefix}/lib64
    includedir=${exec_prefix}/include

    Name: libmfxhw64

    Description: Intel Media SDK dispatcher.
    Version: 2015r6
    Libs: -L${libdir} -lmfxhw64
    Cflags: -I${includedir}

注意这个地方引用的是 `libmfxhw64` 库，因为测试的是 64 位平台。

当然你可以可以选择在 `/usr/lib64/pkgconfig/` 下面创建 `libmfx.pc` 文件。

### mfx/mfxvideo.h

同样是 `libmfx can not found using pkg-config` 这个命令，也可能是头文件的
错误，在 `config.log` 中会报错说无法找到 `mfx/mfxvideo.h` 这个文件。

在安装完 `MMS` 之后，在 `/opt/intel/mediasdk/include/` 目录下面会有
`mfxvideo.h` 这个文件，但是在 `FFMPEG` 中，引用的是 `mfx/mfxvideo.h` 这个
头文件，因此报错，解决的方式是，在 `/opt/intel/mediasdk/include` 这个目录
下面新建目录 `mfx`，然后吧 `include` 的头文件拷贝一份到 `mfx` 目录下。

通过上面这种方式可以修正 `mfx/mfxvideo.h` 无法找到的错误。

# 链接 FFMPEG 时的错误

在链接 `ffmpeg` 的时候还是有可能会出现 `MFXxxx` undefinded reference 的错
误，这时候你需要让你的程序链接到 `lmfxhw64` 这个库。最简单的方式是，在
`/usr/lib64/` 中建立一个 `libmfxhw64` 的软连接

    ln -s /opt/intel/mediasdk/lib64/libmfxhw64.so /usr/lib64/libmfxhw64.so

然后在编译自己的程序的时候加入 `-lmfxhw64` 选项。

# 在 FFMPEG 中使用 qsv 编码器

`qsv` 的编码器在 `FFMPEG` 中有 `h264` 和 `h265` 两种，你可以通过下面的代码
找到这个编码器。

    av_find_encoder_by_name("h264_qsv");

此外，通过 `qsvenc_h264.c` 这个源文件，我们可以看到它支持 `QSV` 和 `NV12`
两种格式，但是 `QSV` 这个格式好像无法正常的使用，你需要把编码的 `pfx_fmt`
设置成 `NV12`。
