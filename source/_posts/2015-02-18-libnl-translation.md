---
author: 郭荣飞
date: 2015-02-18
title: Netlink 库 -- 官方开发者教程中文版目录
tags:
 - netlink
 - libnl
---

这是`libnl`库的官方开发者文档教程的中文翻译，原文可以在[这个地址][source_addr]中
找到。该文档由郭荣飞翻译，希望可以给开源世界贡献一份自己微薄的力量。本人非专业翻
译因此文中必然有翻译不当之处，如果您发现了这样的问题，请联系
<guorongfei@126.com>。

  [source_addr]: http://www.infradead.org/~tgr/libnl/doc/core.html

* * *

目录

1. [引言](/2015/01/20/libnl-translation-part1/#introduction)

  1.1. [如何阅读这份文档](/2015/01/20/libnl-translation-part1/#how_to_read)

  1.2. [如何链接到这个库](/2015/01/20/libnl-translation-part1/#how_to_link)

  1.3. [调试](/2015/01/20/libnl-translation-part1/#debugging)

2. [Netlink 协议基础](/2015/01/23/libnl-translation-part2/#nlp_fundamentals)

  2.1. [寻址](/2015/01/23/libnl-translation-part2/#addressing)

  2.2. [消息格式](/2015/01/23/libnl-translation-part2/#message_formate)

  2.3. [消息类型](/2015/01/23/libnl-translation-part2/#message_type)

  2.4. [序列号](/2015/01/23/libnl-translation-part2/#seq_num)

  2.5. [多播组](/2015/01/23/libnl-translation-part2/#multicast_group)

3. [Netlink 套接字](/2015/01/26/libnl-translation-part3/#netlink_sock)

  3.1. [套接字结构体](/2015/01/26/libnl-translation-part3/#sock_struct)

  3.2. [序列号](/2015/01/26/libnl-translation-part3/#sequence_num)

  3.3. [多播组订阅](/2015/01/26/libnl-translation-part3/#multi_grp_sub)

  3.4. [修改套接字回调配置](/2015/01/26/libnl-translation-part3/#callback_config)

  3.5. [套接字属性](/2015/01/26/libnl-translation-part3/#sock_attri)

4. [消息或数据的发送和接收](/2015/01/27/libnl-translation-part4/#snd_rcv_msg_data)

  4.1. [发送消息](/2015/01/27/libnl-translation-part4/#send_msg)

  4.2. [接收消息](/2015/01/27/libnl-translation-part4/#rcv_msg)

  4.3. [Auto-ACK 模式](/2015/01/27/libnl-translation-part4/#auto_ack)

5. [消息的解析和构造](/2015/01/31/libnl-translation-part5/#message_parse_construct)

  5.1. [消息格式](/2015/01/31/libnl-translation-part5/#msg_formate)

  5.2. [解析消息](/2015/01/31/libnl-translation-part5/#parse_msg)

  5.3. [构建消息](/2015/01/31/libnl-translation-part5/#construct_msg)

6. [属性](/2015/02/12/libnl-translation-part6/#attributes)

  6.1. [属性格式](/2015/02/12/libnl-translation-part6/#attr_formate)

  6.2. [属性解析](/2015/02/12/libnl-translation-part6/#parse_attr)

  6.3. [属性的构造](/2015/02/12/libnl-translation-part6/#attr_construct)

  6.4. [属性数据类型](/2015/02/12/libnl-translation-part6/#attr_datatype)

  6.5. [示例程序](/2015/02/12/libnl-translation-part6/#attr_examples)

7. [回调配置](/2015/02/15/libnl-translation-part7/#callback_config)

  7.1. [回调函数挂钩点](/2015/02/15/libnl-translation-part7/#callback_hooks)

  7.2. [内部函数的覆盖](/2015/02/15/libnl-translation-part7/#overwrite_internal_fun)

8. [高速缓存系统](/2015/02/15/libnl-translation-part8/#cache_system)

  8.1. [高速缓存的分配](/2015/02/15/libnl-translation-part8/#alloc_cache)

  8.2. [高数缓存管理器](/2015/02/15/libnl-translation-part8/#cache_manager)

9. [抽象数据类型](/2015/02/15/libnl-translation-part9/#abstract_data_type)

  9.1. [抽象地址](/2015/02/15/libnl-translation-part9/#abstract_addr)

  9.2. [抽象数据](/2015/02/15/libnl-translation-part9/#abstract_data)
