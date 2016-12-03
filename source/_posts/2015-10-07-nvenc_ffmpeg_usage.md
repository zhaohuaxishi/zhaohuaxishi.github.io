---
author: 郭荣飞
date: 2015-10-07 03:00:00
title: 英伟达硬件加速解码器在 FFMPEG 中的使用
tags:
 - GPU
---

这篇文档介绍如何在 ffmpeg 中使用 nvenc 硬件编码器。

# 私有驱动

`nvenc` 本身是依赖于 `nvidia` 底层的私有驱动的，所以想要使用编码器首先需要
安装 `nvidia` 的私有驱动。在 [NVIDIA VIDEO CODEC SDK][nvenc_doc] 的介绍中
说明，最新版本的 `nvenc sdk 5.0` 在 linux 需要 346.22 以上的驱动，在
windwos 下则需要 347.07 以上的驱动

<!--more-->

> The latest NVENC SDK version available is 5.0, which requires NVIDIA GPU
> driver 347.09 or above for Windows and 346.22 or above for Linux.

目前 Ubuntu 15.04 上的驱动满足这个要求，Windows 平台可以直接到官网上下载最
新的驱动安装。（个人不建议去官网下载最新的 Linux 驱动，因为我试了很多次都
没有安装成功，最终会导致无法进入系统）。

在 Ubuntu 15.04 下使用下面的命令安装最新的驱动。

    sudo apt-get install nvidia-346 \
                         nvidia-346-vum \
                         nvidia-modprobe \
                         nvidia-opencl-icd-346 \
                         nvidia-prime \
                         nvidia-settings

注意 `nvidia-modprobe` 必须要安装，因为私有驱动使用的内核模块，需要安装这
个包在系统启动的时候加载这些内核模块。安装完成之后可能无法进入系统，这个应
该是 `nvidia` 中的一个 `BUG`，你可以重启之后选择 `grub` 中的 `ubuntu 高级
选项` 中低版本的内核进入系统之后重启再选择高版本的内核进入系统。这一点非常
的诡异，目前没有找到原因。

启动系统之后使用 `lsmod | grep nvidia` 应该会得到类似下面的结果：

    nvidia_uvm             69632  0
    nvidia               8380416  36 nvidia_uvm
    drm                   348160  7 i915,drm_kms_helper,nvidia

直接通过 `sudo modprobe nvidia_uvm` 好像也无法成功的加载需要的模块。

另外安装驱动安装完成之后会在 /dev 下面创建几个和 `nvidia` 相关的设备，通过
`ls /dev/nvidia*` 应该会得到类型以下的结果：

    /dev/nvidia0  /dev/nvidiactl  /dev/nvidia-uvm

# 编译 FFMPEG

要想在 FFMPEG 中使用 `nvenc` 编码器，你需要在编译选项中加入 `enable-nvenc`
选项。这个选项依赖于 `nvEncodeAPI.h` 头文件，这个头文件并没有包含在私有驱
动中，你需要到 [NVIDIA VIDEO CODEC SDK][nvenc_doc] 中下载 SDK，解压后在
`Samples/common/inc` 目录下有这个头文件，把它拷贝到可以链接到的目录中去。

之后编译就可以顺利的通过，得到包含 `nvenc` 编码器的库。

# 使用 nvenc

FFMPEG 中直接使用 `av_find_encoder_by_name("nvenc")` 就可以找到这个这个编
码器并使用它。`nvenc.c` 的 `pix_fmts_nvenc` 变量定义来看，这个编码器应该是
支持 `YUV420P`, `YUV444P` 和 `NV12` 三种格式的，但是测试的过程中发现
`YUV420P` 没办法使用，所以应该吧 `AVCodecContext` 的 `pix_fmt` 设置成
`NV12`。

[nvenc_doc]: https://developer.nvidia.com/nvidia-video-codec-sdk
