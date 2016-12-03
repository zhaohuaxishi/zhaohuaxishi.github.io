---
author: 郭荣飞
date: 2015-10-07 02:00:00
title: 英伟达硬件解码器分析
tags:
 - GPU
---

这篇文章主要分析 NVCUVID 提供的解码器，里面提到的所有的源文件都可以在英伟
达的 `nvenc_sdk` 中找到。

# 解码器的代码分析

SDK 中的 sample 文件夹下的 NvTranscoder 中包含了编码器和解码器的用法，编码
器的内容不在这里分析，因为 FFMPEG 中已经包含了相关的代码，不需要其他的处理
。

解码器在 SDK 中有一份封装，主要是 NvTranscoder 下的 VideoDecoder 类。目前
这个类的具体用法还不是特别的清楚。分析将会从 main 函数开始。

<!--more-->

# main

NvTranscoder 有一个单独的文件，执行逻辑从 `main()` 函数开始。

cuInit(0) 应该是初始化 cuda 的相关代码。目前没有找到定义，估计类似于
FFMPEG 中的 `av_register_all`。

105 行之前的代码是编码相关的代码， 85 行以前设置各种编码参数，87 行分析从
命令行读入的参数，100 行打开目标输出文件。

110 行开始代码应该是和解码相关的核心代码。`cuDeviceGet`, `cuCtxCreate`
`cuCtxPopCurrent`, `cuvidCtxLockCreate` 应该是固定写法。初始化一些内部机制
。

120 行的 `InitVideoDecoder` 是解码器创建的地方。这个函数需要认真的分析，整
个解码器的关键代码应该就在这里面。

137 行初始化 119 行中创建的 FrameQueue。这个队列应该相当于解码器和编码器之
间的一个缓冲区，解码器方内容进去，编码器从中取内容出来。

139 行到 180 都是关于编码器的设置。这里不做详细的分析。

187 行的注释显示 `pthread_create` 创建出来解码线程，所以解码工作是由这个线
程完成的。线程执行的函数是 `DecodeProc` 这个函数，而这个函数只不过是调用的
解码器的 `Start()` 方法。

195 行的代码注释来看，这后续的代码都是编码相关的代码，这里不做分析。最后在
245 行和 246 行的未知 `cuvidCtxLockDestroy` 和 `cuCtxDestroy` 应该是对应于
110 行的那些代码。

# VideoDecoder.cpp

这个文件实现了解码器的封装类 CudaDecoder, 这个类是整个硬件解码器实现的关键
，但是这个类其实比较简单。它在 `main` 函数中涉及到的方法只有
`InitVideoDecoder`, `GetCodecParam`, `Start` 和 `GetDecoder` 这四个。

## InitVideoDecoder

这个方法是首先被调用的方法。它负责编码器的初始化操作。

从 VideoDecoder.cpp 中的实现来看，初始化主要包括三个部分：

### 创建视频源

视频源的参数是 CUVIDSOURCEPARAMS，其中设置了一个 `pfnVideoDateHandler`，从
字面上理解它是一个视频数据的回调处理函数。

创建视频源的方法是 `cuvidCreateVideoSource()` 函数，目前来说这个函数的致命
问题在于它的接收 videoPath 作为参数，这似乎意味着它只能处理文件视频源。这
个函数的函数原型定义在 `nvcuvid.h` 这个头文件中，这个头文件只定义了下面这
些和视频源相关的接口：

    cuvidCreateVideoSource();
    cuvidCreateVideoSourceW();
    cuvidDestroyVideoSource();
    cuvidSetVideoSourceState();
    cuvidGetVideoSourceState();
    cuvidGetSourceVideoFormat();
    cuvidGetSourceAudioFormat();

从接口来看只有 `cuvidCreateVideoSource(); cuvidCreateVideoSourceW();` 这两
个函数可用，而它们唯一的区别在于接收不同的文件路径字符串，前者是普通字符而
后者是宽字符。

目前暂时没有其他的资料表明可以创建非文件类型的视频源，所以这个解码器的用处
估计不会太大，至少在传屏应用中的用处会相对较小。

### 获取视频源的参数，并创建 cuvid 库的解码器

视频源的参数信息在创建视频源之后可以通过 cuvidGetSourceVideoFormat() 函数
获得，在 `InitVideoDecoder()` 函数中获取参数最诡异的地方在于 111 行创建了
一个 CUVIDOFORMATEX 类型的变量 `oFormatEx`，然后让 `oFormat` 引用这个变量
的 format 字段。在调用 `cuvidGetSourceVideoFormat` 之后竟然可以直接访问
`oFormatEx` 这个变量的 `raw_seqhdr_data` 字段，个人估计它的内部实现使用了
类似 `container_of` 这样的技术访问了 `oFormatEx`。但是为什么这样设计不得而
知。

创建解码器的函数是 `cuvidCreateDecoder()`，这个函数的原型定义在 cuviddec.h
文件中。

    cuvidcreatedecoder(cuvideodecoder *, CUVIDDECODECREATEINFO *);
    cuvidDestroyDecoder(CUvideodecoder);

解码器的参数是通过 `CUVIDDECODECREATEINFO` 传递的，这个结构体的大部分字段
都是通过前面获得的 `oFormat` 中的信息获得。

### 创建视频源的解析器

初始化的最后一步是创建一个视频源的解析器，其中设置了三个回调函数，
`HandleVideoSequence`, `HandlePictureDecode`, `HandlePictureDisplay`, 在
`nvidia` 的文档中并没有说这些回调函数会在什么时候调用，也没有说明这些回调
函数要完成的事情是什么，只能从名字中猜测 `HandlePictureDecode` 这个函数是
用来解码的。

# Start

在 `main` 函数的解码线程函数中只调用了 CudaDecoder 类的 Start 函数。而
`Start` 函数本身也非常的简单，只不过调用了 cuvidSetVideoSourceState() 把状
态变成 cudaVideoState_Started 然后一直取状态直到状态不再是 `started`。

从这个函数的实现来看，它的内部应该在把视频源设置为
`cudaVideoState_Started` 状态之后开始读取视频源（文件）中的数据。然后通过
回调函数进行处理。应该是首先调用 `HandleVideoData()`, 个人猜测这个函数在数
据从文件中读取出来之后会被调用来解析原始数据，`HandleVideoSequence` 这个函
数没有太大的用途，只是一些参数的检测而已。`HandlePictureDecode` 应该是在成
功解析到数据帧的时候调用，这个函数调用了 `cuvidDecodePicture` 解码数据。数
据解码出来之后会调用 `HandlePictureDisplay` 函数，该函数把数据放入到数据缓
冲区 `FrameQueue` 中以便编码器能够把数据取出来。

# 总结

使用 CudaDecoder 首先需要调用 cuvidCreateVideoSource 创建一个文件视频源，
然后调用 `cuvidGetSourceVideoFormat` 从文件中读取解码参数信息并使用参数信
息创建一个 `CUvideodecoder` 解码器，之后再创建一个视频源解析器，设置回调函
数处理视频的解码。

上面的初始化完成之后调用 `cuvidSetVideoSourceState` 把视频源的状态设置为
`cudaVideoState_Started`，之后库的内部会开始读文件，把读取的数据交给
`HandleVideoData` 解析，解析完成之后会把数据交给 `HandlePictureDecode` 调
用 `cuvidDecodePicture` 进行解码。在解码完成之后调用
`HandlePictureDisplay` 把数据放入到 FrameQueue 缓冲区里面。

# 补充：video source 和 nvcuvid

在 SDK 给出的例子中，数据是通过 `video source` 接口来提供的。但是这并不意
味着我们在编写程序的时候只能使用它提供的 `video source` 接口。根据[官方文
档 ][nvcuvid_doc]中第三小节最后给出的解释

>  Note: The low level decode APIs are supported on both Linux and Windows
>  platforms. The NVCUVID APIs for Parsing and Source Stream input are
>  available only on Windows platforms.

NVCUVID 的 `video source` 只在 `windows` 平台可用，不过从最新的
`nvenc_sdk` 的代码来看，videosource 和 sourcepraser 在 linux 平台下也是可
用的。只不过从接口来看，这两个 API 只能用于文件的解析。

在文档的第四小节 4.2 中有这么一段话：

> For Linux platforms, you will need to write your own video source and
> parsing functions that connect to the Video Decoding functions.

这一点明确说明，其实我们可以不使用它本身的 `video source` 接口，使用自己的
接口提供视频源，然后使用 `nvcuvid` 最底层的解码接口对数据进行解码和后续处
理。

# MAP

在 nvcuvid 的官方文档中给出的接口中，最诡异的两个接口是

    cuvidMapVideoFrame()
    cuvidUnmapVideoFrame()

这两个函数好像是用于处理解码之后的数据的，但是这其中的原理是什么并不清楚，
有待后续研究。

[nvcuvid_doc]: http://docs.nvidia.com/cuda/video-decoder/#axzz3mWQbRpKv
