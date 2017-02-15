---
title: 捉妖记之——参数求值顺序
date: 2017-02-11 07:49:12
tags:
 - 捉妖记
---

2017-02-10 碰到一个参数求值顺序导致的程序崩溃问题，花了几个小时去解这个问题，觉
得非常有意思，记录在此。

<!--more-->

# 场景

为了完成一个网络传输数据的功能，我定义了一个类表示数据包，包含`包头`和`数据`两个
字段（忽略这里的 public 成员变量，我只是想让这篇文章写起来简短些）：

```c++
class Packet {
public:
    PacketHeader header;
    std::vector<uint8_t> data;
}
```

因为数据有不同的类型，所以我在类的内部定义了一个枚举：

```c++
class Packet {
public:
    enum class Type {
        kHeartBit,
        kBitStream
    };
private:
    PacketHeader header;
    std::vector<uint8_t> data;
}
```

数据包头包含类型和数据长度两个字段：

```c++
struct PacketHeader {
    using VideoType = Packet::Type:
    using SizeType = std::vector<uint8_t>::size_type;

    PacketHeader(VideoType type, SizeType len);

    VideoType type;
    SizeType len;
};
```

为了方便数据包，我提供了N个重载的构造函数：


```c++
class Packet {
public:
    Packet(Type type, const std::vector<uint8_t>& data);
    Packet(const PacketHeader& header, const std::vector<uint8_t>& data);
private:
    PacketHeader header;
    std::vector<uint8_t> data;
}
```

为了能够尽可能的减少重复代码，我使用了委托构造函数来实现第一个构造函数。

```c++
Packet::Packet(Type type, const std::vector<uint8_t>& data)
    : Packet(PacketHeader(type, data.size()), data) {}
```

最后为了传输，我定义了一个函数用于`Packet`的序列化：

```c++
std::vector<uint8_t> Serialize(const Packet& packet) {
    if (IllegalPacket(packet)) {
        return vector<uint8_t>();
    }

    std::vector<uint8_t> bitstream(sizeof(PacketHeader) + packet.header.len);

    // 拷贝头部
    ...

    // 拷贝数据
    auto data = std::begin(bitstream);
    std::advance(data, sizeof(PacketHeader));
    std::copy(std::begin(packet.data), std::end(packet.data), data);
}
```

上面这段代码起初工作的很不错，单元测试正常的运行，一切看上去都很顺利，直到我觉得
我需要支持指针形式和右值形式的数据构造，于是我添加了下面这两个`Packet`构造函数：

```c++
class Packet {
public:
    Packet(Type type, const std::vector<uint8_t>& data);
    Packet(Type type, uint8_t*data, int len);
    Packet(Type type, std::vector<uint8_t>&& data);

    Packet(const PacketHeader& header, const std::vector<uint8_t>& data);
    Packet(const PacketHeader& header, uint8_t*data, int len);
    Packet(const PacketHeader& header, std::vector<uint8_t>&& data);
private:
    PacketHeader header;
    std::vector<uint8_t> data;
}
```

按照《Effective Modern C++》中的建议，上面这些构造函数可以统一成值传递的形式，然
后以 move 的形式完成数据的构造，因此我把代码实现成了下面这个样子：


```c++
class Packet {
public:
    Packet(Type type, std::vector<uint8_t> data)
        : Packet(PacketHeader(type, data.size()), std::move(data)) { }

    packet(const packetheader& header_, std::vector<uint8_t> data_)
        : header(header_), data(std::move(data_)){}
private:
    PacketHeader header;
    std::vector<uint8_t> data;
}
```

正在我暗自欣喜能学以致用的时候，我运行了一下单元测试，很遗憾的是，程序崩溃了。因
为了我使用了`GUN STL`的`DEBUG`模式，运行时报错说我的`vector`迭代器越界了。

# 问题分析

当时项目比现在看到的复杂的多，很多地方都使用了`vector`，可惜`DEBUG`模式的越界输
出中并说到底是迭代器越界了，更坑爹的是它不会输出`coredump`。我通过人工追踪调用栈
，最终定位到问题出现在序列化函数`Serialize`函数中。我们再看看这个函数：

```c++
std::vector<uint8_t> Serialize(const Packet& packet) {
    if (IllegalPacket(packet)) {
        return vector<uint8_t>();
    }

    std::vector<uint8_t> bitstream(sizeof(PacketHeader) + packet.header.len);

    // 拷贝头部
    ...

    // 拷贝数据
    auto data = std::begin(bitstream);
    std::advance(data, sizeof(PacketHeader));
    std::copy(std::begin(packet.data), std::end(packet.data), data);
}
```

这个函数中唯一可能出现迭代器越界可能的就是`std::copy`中的`data`参数写入越界了！
但是为什么会越界呢？bitstream的长度是头部数据加上数据长度啊，而且我在操作之前已
经确保了这是一个合法的包！而且为什么只是把参数改成了move之后就不行了呢？到底什么
地方出了问题，我想了又想，我猜了又猜，这个问题还真奇怪。

我花了将近半个小时的时候去纠结这个问题始终找不到问题的答案，最后我决定把整个数据
包的信息打印出来，这才发现了问题的所在，我得到了下面的数据：

```
type: bitsteam, len: 0, data_size: 1
```

## 第一个 BUG

没错，我的包头中的数据长度和实际的数据大小是不一致的。也就是说，我的
`IllegalPacket`函数写的有问题。因为数据包可能是序列化之前的数据包也可能是序列化
之后网络传输的数据包。说以对于这个函数我有两种形式的重载：

```c++
bool IllegalPacket(const std::vector<uint8_t>& bitstream);
bool IllegalPacket(const Packet& packet);
```

因为网络传输的数据包可能并不完整，所以对于第一个函数我的写法如下：

```c++
bool IllegalPacket(const PacketHeader& header, int len) {
    if (IllegalPacketHeader(header)) {
        return true;
    }

    return header.len > len;
}

bool IllegalPacket(const std::vector<uint8_t>& bitstream) {
    PacketHeader hader = GetHeader(bitsrteam);
    return IllegalPacket(header, bitsrteam.size() - sizeof(PacketHeader));
}
```

也就是说只要数据的包头合法而且`bitstream`中的长度不小于包的实际长度（包头长度+包
头中的数据长度）我就判断为合法。

为了尽可能的复用代码，我把第二个函数写成了下面这个样子：

```c++
bool IllegalPacket(const Packet& packet) {
    return IllegalPacket(packet.header, packet.data.size());
}
```

我确实尽我最大的努力复用了代码，而且起初这些代码初看上去没什么问题。但是我终究还
是坑死在自己的手上，我忘记了编码的基本原则之一：“clean code that works”，这段代
码虽然 clean，但是它不 work。因为`std::vector<uint8_t>`形式的包和`Packet`形式的
包的合法性判断条件并不相同，它们的本来就不应该复用同一个判定逻辑！对于一个
`Packet`形式的包来说，只要`header.len != data.size()` 那么它就不是一个合法的包。
所以第一个应该改的地方是

```c++
bool IllegalPacket(const Packet& packet) {
    if (IllegalPacketHeader(packet.header)) {
        return true;
    }

    return packet.header.len != packet.data.size();
}
```

所以其实上面两个同名为`IllegalPacket`的函数本身的语义就不太一样，为了代码阅读的
便利性，我把第一个函数重命名为`ContainLegalPacket`


```c++
bool ContainLegalPacket(const std::vector<uint8_t>& bitstream) {
    PacketHeader hader = GetHeader(bitsrteam);
    if (IllegalPacketHeader(header)) {
        return false;
    }

    return header.len <= len;
}
```

## 第二个BUG

解决了上面这个BUG之后，重新编译运行单元测试，程序没有崩溃了，但是单元测试依旧没
有通过，因为`Serialize`函数的函数输出和我的预期不相符了。

下面这段单元测试无法通过：

```
std::vector<uint8_t> data = {1};
Packet packet(Packet::Type::kBitStream, std::move(data));
REQUIRE(!Serialize(packet).empty());
```

日志显示，我传入的包是非法的包，也就是说我用构造函数构建的包是非法的包。这说明构
造函数有问题，这就回到了最初改动之后引发崩溃的地方，也就是下面这个构造函数。

```c++
Packet(Type type, std::vector<uint8_t> data)
    : Packet(PacketHeader(type, data.size()), std::move(data)) { }
```

它实际调用就的是这个构造函数：

```c++
Packet(const PacketHeader& header_, std::vector<uint8_t> data_);
```

给你三秒钟，想想上面的代码有什么问题。这是一段为了代码复用+效率最大化而编写的代
码，出发点是没有问题的，但是它的实际效果和你预想的并不一致。

时间到了，答案是参数有问题，更确切的说是参数的求值顺序的问题。C++并规定参数的求
值顺序，但是GCC通常是使用从右往左的构造顺序。它先求值了第二个参数，然后是第一个
参数，如果你展开了这个求值过程，伪代码大概如下：

```c++
std::vector<uint8_t> data_ = std::move(data);
PacketHeader temp(type, data.size());
const PacketHeader& header_ = temp;
```

现在BUG应该是一目了然的，`move`了`data`，随后却调用了它的`size()`成员函数，理论
上这是一段未定义行为的代码，因为`move`之后的对象处于可析构的状态，但是你不应该在
调用它的任何成员函数。

解决这个问题的办法很简单：放弃委托构造。

```c++
Packet(Type type, std::vector<uint8_t> data_)
    : header(type, data.size()), data(std::move(data_)) { }
```

## 额外的一个问题

其实如果我们回头看最初崩溃的地方你就会发现，这里的代码有一个前后逻辑不一致的地方

```c++
std::vector<uint8_t> Serialize(const Packet& packet) {
    ...
    std::vector<uint8_t> bitstream(sizeof(PacketHeader) + packet.header.len);
    ...
    std::copy(std::begin(packet.data), std::end(packet.data), data);
}
```

这里我构造 bitstream 的时候用的是`packet.header.len`但是实际拷贝数据的时候使用的
长度是`std::distance(std::begin(data), std::end(packet.data))` 也就是
`packet.data.size()`。这段代码能正常工作的前提就是`packet.header.len =
packet.data.size()` 但是这个前提条件却没有在代码中做额外的检查，所有更合理的方式
应该是：

```c++
std::vector<uint8_t> Serialize(const Packet& packet) {
    ...
    assert(packet.header.len == packet.data.size());
    std::vector<uint8_t> bitstream(sizeof(PacketHeader) + packet.header.len);
    ...
    std::copy(std::begin(packet.data), std::end(packet.data), data);
}
```

或者在构造的时候干脆就直接使用：`sizeof(PacketHeader) + packet.data.size()`

```c++
std::vector<uint8_t> Serialize(const Packet& packet) {
    ...
    std::vector<uint8_t> bitstream(sizeof(PacketHeader) + packet.data.size());
    ...
    std::copy(std::begin(packet.data), std::end(packet.data), data);
}
```

我更加趋向于第一种做法，因为从输出的角度考虑，序列化函数输出的数据长度本来就应该
是包头长度加上包头中`len`字段的长度，因为这是解序列化函数的内在逻辑。

# 反思

这是一个非常有趣的BUG，我从中得到下面这教训：

- 正确的使用函数的重载，IllegalPacket 就是应该例子，两个重载函数的内在逻辑并不一
  致，但是同样的名字误导我认为它们一致，并且重用判断逻辑

- 先保证代码的正确，再考虑代码的效率。让正确的代码提高效率，比让高效率的代码正确
  难很多。比如 std::move

- 小心非法的迭代器访问，比如 std::copy

- 想清楚你写的函数的合约（前后置条件，不变量等），多使用 assert 表示你的合约
