---
title: 捉妖记之——成员变量的析构顺序
date: 2017-02-19 11:29:05
tags:
 - 捉妖记
---

受RAII思想影响比较深的我，一直认为如果程序组织的好，你通常是不需要编写析构函数的
（析构通常用于资源的释放，如果你的类使用的资源都是RAII管理的话，通常也就不需要自
己编写析构函数了）。但是因为过于迷恋这一点，2017-02-15我写出了一个坑了很久的BUG
。

<!--more-->

# 场景

我需要在Linux平台下录制桌面，我使用了`ffmpeg`的`x11grab_xcb`，我需要渲染出我录制的数
据，我使用了`SDL2`进行渲染。

我单独测试了`ffmpeg`的代码，和`SDL2`的代码，一切运行起来都比较正常，但是当我把这
两个模块的代码，组合起来之后，程序崩溃了。代码结构很简单，大致如下图：

```
+-----------+          +------------+
|  Sender   |          |  Receiver  |
|-----------|          |------------|
|           | network  |            |
|  x11grab  +---------->   SDL2     |
|           |          |            |
+-----------+          +------------+
```

崩溃的消息很明确：

```
[xcb] Unknown request in queue while appending request
[xcb] Most likely this is a multi-threaded client and XInitThreads has not been called
[xcb] Aborting, sorry about that.
lt-catch: ../../src/xcb_io.c:165: append_pending_request: Assertion `!xcb_xlib_unknown_req_pending' failed.
```

# 问题分析

从错误消息来看，很明显是 `xcb` 内部出了问题，当时我想都没有像，就直接去查
`sender`相关的代码，因为它使用了`x11grabe_xcb`进行屏幕的录制，我用了我知道的各种
调试方法：google，gdb，coredump，log，二分。调试的过程非常的漫长，此处省略一万字
。然而我最终还是一无所获，直到我开始怀疑人生，开始怀疑我调试的方向就是错的。

## 南辕北辙

武行四大忌：道士和尚女人小孩，DEBUG的第一大忌我觉得是南辕北辙。这个BUG我解了大半
天之后才开始怀疑自己的最开始的方向是错的。`sender`很明显使用了`x11grab_xcb`，但
是实际上的BUG却不见得一定是它引起的。意识到这一点之后，我把目光转向了`Receiver`
，也就是`SDL2`上。

后来我发现，其实`SDL2`这个库在`Linux`平台下也是依赖于`xcb`的，而问题也正是处在
`SDL2`这个库的使用上！

## 自动析构的顺序

`SDL2`中的大部分数据结构都有一个`create`和一个对应的`destroy`，也就是说你必须在
合适的时候调用`destroy`来释放你分配的资源，为了自动化管理这些资源的生命周期，我
使用了`std::shared_ptr`

```c++
class SDLRenderer {
...
private:
    std::shared_ptr<SDL_Texture> texture_;
    std::shared_ptr<SDL_Renderer> texture_;
    std::shared_ptr<SDL_Windows> texture_;
};
```

此外为了执行`SDL2`的事件，我单独起了一个线程，听取了《Effective Modern C++》的建
议我使用了`std::future<void>`也就是`task-base`的`std::async`来创建线程。

```c++
class SDLRenderer {
private:
    std::future<void> mainloop_;
    std::atomic_bool running_;
    ...
};

SDLRenderer::SDLRenderer() {
    running_ = true;
    mainloop_ = std::async(std::launch::async, [this] {
        while(running_) {
            SDL_PollEvent(nullptr);
            std::this_thread::sleep_for(std::chrono::seconds(1));
        }
    });
}
```

为了能够，让`mainloop_`停下来，我需要在析构函数中把`running_`设置成`false`。也就
是说我需要手动写一个析构函数：

```c++
SDLRenderer::~SDLRenderer() {
    running_ = false;
}
```

我并没有在析构函数中处理其他的数据成员，因为其他的成员默认析构行为都是符合预期的
：`std::shared_ptr`会自动释放我分配的资源，`std::future`在这种情况下会自动调用
`get`（std::future什么情况下会自动调用`get`可以参考《Effective Modern C++》一书
）。

上面的代码还无法正常的运行，因为我们没有调用`SDL_Init`，和`SDL_Quit`，这两个函数
必须在`SDL2`的其他函数调用的首尾调用（`SDL_Init`在其他`SDL2`函数之前，而
`SDL_Quit`在其他`SDL2`函数之后，当然这种说法不是很准确，文档中没有直接这么说，但
是应该是一种使用惯例）。很显然最合适的调用点就是在构造和析构函数中，所以有了下面
的代码：

```c++
SDLRenderer::SDLRenderer() {
    SDL_Init();
    ... // 初始化 mainloop_ 的代码同上
}

SDLRenderer::~SDLRenderer() {
    running_ = false;
    SDL_Quit();
}
```

# BUG

上面的代码正是引起程序崩溃的原因，问题出在析构函数里面。这个函数实际上的代码并不
是我们看到的这个样子，因为编译器会自动插入成员变量的析构函数的调用，这些析构函数
里面实际上会调用相关的资源释放函数，也就是说实际上的析构函数代码，大概是这个样子
的：

```
SDLRenderer::~SDLRenderer() {
    running_ = false;
    SDL_Quit();

    SDL_DestroyWindow(window_.get());
    SDL_DestroyRenderer(renderer_.get());
    SDL_DestroyTexture(texture_.get());
    SDL_PollEvent(nullptr);
}
```

上面这段代码一看就知道函数的调用顺序应该是有问题的。所以解决方式也比较简单，我们
有两种方式可以解决这个问题，第一种就是手动调整成员变量的析构顺序：

```
SDLRenderer::~SDLRenderer() {
    running_ = false;
    mainloop_.get();

    texture_.reset(nullptr);
    renderer_.reset(nullptr);
    window_.reset(nullptr);

    SDL_Quit();
}
```

这样一来，`SDL2`相关的函数调用顺序就得以解决，另外一个方式是，将`RAII`进行到底，
写一个`RAII`管理`SDL_Init()`和`SDL_Quit()`，然后直接省略析构函数。这里就不罗列相
关的代码了。

# 反思

- 确认成员函数的析构顺序，即使你使用了RAII，RAII确实可以自动化管理你的成员变量的
  生命周期，但是这并不代表它可以自动化你的成员变量的析构顺序。

- 要开自动挡的车！老司机习惯自动挡的车，因为喜欢控制一切的快感；马路杀手还是开自
  动挡的车靠谱些；无论你是老司机还是马路杀手都不要来回在自动挡和手动挡之间切换，
  迟早要出事儿的，资源管理也是这个道理。
