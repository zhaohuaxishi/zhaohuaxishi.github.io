---
title: 捉妖记之——状态机模式
date: 2018-12-09 13:56:56
tags:
 - 捉妖记
 - C/CPP
---

状态机是一种非常好的表达逻辑跳转的方式，但是本身容易导致代码的耦合，维护性不佳。状态机模式是状态机结合面向对象之后的产物，既保持了状态机，清晰的逻辑表达能力，又加入了面向对象的高内聚性和可扩展性，提升代码的可维护性，是我非常喜欢的一种设计模式。

最近碰到一个问题，比较适合用状态机来描述，果断的使用了状态机模式，但是编写代码的过程中出现了一个隐藏很深的BUG，记录在这里，方便复习。

# 背景

最近在做网络编程，网卡会有开关，我需要在网卡打开的时候创建对象，关闭的时候销毁对象，但是创建可能失败，这个时候，我需要不断的重试，直到成功为止。我把这个问题分成了三个状态：stop、ready、runing，他们之间的转换关系如下：

```
+-------+    up      +-------+
|  stop |<---------->| ready |
+-------+   down     +-------+
    ^                    |
    | down               | create success
+--------+               |
| runing |<--------------+
+--------+
```

`stop`状态手动网络UP消息的时候会转换为`ready`状态，准备创建对象，如果对象创建失败会一直在`ready`状态，直到创建成功之后，转换为`runing`状态。

# 编码实现

## 抽取基类

我们抽象出公共的状态`State`，里面有`OnUp`和`OnDown`两个函数：

```cpp
std::unique_ptr<state> state;

class State {
public:
    virtual ~State() = default;

    virtual void OnUp() = 0;
    virtual void OnDown() = 0;
};
```

## 编写子类

实现这个对象的三个子类：

```cpp
class StopState : public State {
public:
    virtual void OnUp() {
        state.reset(new ReadyState);
    }
};

class ReadyState : public State {
public:
    ReadyState() {
        CreateWidget()
    }

    void CreateWidget() {
        try {
            // create widget
            state.reset(RunningState);
        } catch(...) {
            TrayAgainLater();
        }
    }

    void TrayAgainLater() {
        // set timer and after timout, call CreateWidget again
    }

    virtual void Down() {
        state.reset(new StopState);
    }
};

class RunningState : public State {
    virtual void Down() {
        state.reset(new StopState);
    }
};
```

到目前为止，上面的代码都能正常的work，直到我们的最后一步，初始化状态。

## 初始化状态

我们需要根据初始状态下，网卡是开还是关来判断是初始化成为`stop`状态还是`ready`状态。

```cpp
void Init() {
    if (network_adaptor.IsUp()) {
        state.reset(new ReadyState);
    } else {
        state.reset(new StopState);
    }
}
```

逻辑上来说，这段代码似乎没有什么问题，但测试用例死活无法通过，我最终通过打印追踪发现原来我的状态一直停留在`ReadyState`。

# 问题分析

上面这段代码，逻辑上没有问题，正在的问题出现在最后一步，状态初始化的地方：

```cpp
state.reset(new ReadyState);
```

按照我们的逻辑，我们设置初始化状态为`ready`，然后程序开始构造对象，并最终在成功构造之后一直停留在`runing`状态。但是函数调用是栈式调用而不是顺序调用，展开后的函数调用栈如下：

```cpp
ReadyState::ReadyState();
    ReadyState::CreateWidget();
        RunningState::RunningState();
        unique_ptr::reset(runing);
unique_ptr::reset(ready);
```

从这个调用栈中，我们可以发现我们确实切换到了`running`状态，但是很快我们又回到了`ready`，这里的根本问题在于，我们`CreateWidget`这个函数的调用放在了`ReadyState`的构造函数中，从而导致`RunningState`仅仅存在了非常短的一段时间。

理想中，我们的函数调用顺序应该是这样的：

```cpp
ReadyState::ReadyState();
unique_ptr::reset(ready);
ReadyState::CreateWidget();
    RunningState::RunningState();
    unique_ptr::reset(runing);
```

这样我们最终就留在了`running`状态中，所以解决这个问题的方案是把`CreateWidget`的调用挪出来。

```cpp
class ReadyState : public State {
public:
    ReadyState() {
        // 不调用 CreateWidget
    }
};
```

```cpp
void Init() {
    if (network_adaptor.IsUp()) {
        ChangeToReadyState()
    } else {
        ChangeToStopState()
    }
}

void ChangeToReadyState() {
    auto ready = new ReadyState();
    state.reset(ready);
    ready->CreateWidget();
}
```

# 反思

尽量避免在状态的构造函数中切换到另外一个状态
