---
author: 郭荣飞
date: 2014-09-30
title: C 语言的趣味用法
tags:
 - C/CPP
---

# 零长数组——Zero-Length Array

``` c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

struct packet_head {
        int  len;
        char content[0];
};

int main(void)
{
        char content[] = "hello world";
        struct packet_head *packet;

        packet = malloc(sizeof(struct packet_head) + sizeof(content));
        packet->len = sizeof(content);

        memcpy(packet->content, content, sizeof(content));
        printf("len: %d, content: %s\n", packet->len, packet->content);

        return 0;
}
```

在上面这段代码中结构体 `packet_head` 有一个成员 `content` 是一个长度为 0 的数组
。这种用法在 C 标准中并不存在，它是 GCC 的一个扩展用法。在编写网络协议实现的时候
用的比较多，在包头的结构体末尾定义一个这样的成员可以使得它指向包头后面的数据部分
。比如在上面的例子中，`malloc` 分配了包头和数据的空间，把返回值转换成包头类型，
从而使用 `packet->content` 来控制数据部分。

<!--more-->

# 结构体内部的宏定义


``` c
struct __wait_queue {
    unsigned int        flags;
#define WQ_FLAG_EXCLUSIVE    0x01
    void               *private;
    wait_queue_func_t   func;
    struct list_head    task_list;
};
```

宏定义本身是没有作用域限制的，把宏定义写在一个结构体的内部并不表示这个这个宏定义
只能用在这个结构体内部。宏定义在编译之前的预处理部分就已经被替换了。之所以把宏定
义写在一个结构体的内部一般是表示这个宏定义一般用于结构体的某个成员，比如上面的这
个结构体中的宏定义 `WQ_FLAG_EXCLUSIVE` 用来表示结构体中 flags 的值（互斥的唤醒）
。


# 结构体内部的匿名联合体（或者结构体）


``` c
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    enum person_type {
            STUDENT,
            TEACHER
    };

    struct person_struct {
            enum person_type type;
            union {
                    int student_age;
                    float teacher_salary;
            };
    };

    struct person_struct student = {
            .type = STUDENT
    };

    struct person_struct teacher = {
            .type = TEACHER
    };

    student.student_age = 23;
    teacher.teacher_salary = 6543.21;

    printf("Student age: %d\n", student.student_age);
    printf("Teacher salary: %.2f\n", teacher.teacher_salary);

    return 0;
}
```


在上面的例子中，`person_struct` 这个结构体内部包含了一个匿名的联合体，你可以通过
`person.student_age` 或者 `person.teacher_salary` 来引用这两个值。这是 C11 中的
新特性。


# 对齐


``` c
#define SKB_DATA_ALIGN(X)    (((X) + (SMP_CACHE_BYTES - 1)) & \
                             ~(SMP_CACHE_BYTES - 1))
```


另 `SMP_CACHE_BYTES` - 1 为 mask 则上面的宏定义简化成 （x + mask）& （～mask），
首先加上 mask 使得不够 `SMP_CACHE_BYTES` 的部分先补齐，然后 & ～mask 其实就是把
数据不足 `SMP_CACHE_BYTES` 的部分清除掉。比如 `SMP_CACHE_BYTES` = 4 （100），那
么 mask = 3 （011）。假设 x = 3，那么最后的结果是 110（6 = 3 + 3）& 100（～3）=
100（4）。如果 x = 1，那么结果就是 100（4 = 1 + 3） & 100 （～3） = 100（4）。假
设 x= 5，那么最后的结果是 1000（8 = 5 + 3） &（1100） =  1000（8）。假设 x = 7，
那么结果是 1010（10 = 7 + 3） & 1100（～3） = 1000 （8）。


# 强迫编译时出错


``` c
#define BUILD_BUG_ON(condition) ((void)sizeof(char[1 - 2*!!(condition)]))
```


这个宏定义有两个地方值得注意：第一就是 !!(condition) 的使用，我们知道在 C 语言中
使用 非0 表示真，0表示假（而不是 1 表示真，0 表示假）。如果你想强制使用 1 表示真
，0 表示假的话可以使用 !!(condition)。因为两次使用逻辑否运算不会改变 condition
到逻辑属性，但是因为逻辑运算到结果在 c 语言中用 1 表示真，0 表示假，所以最后到结
构可以保证一定使用 1 表示真，0 表示假。

第二个值得注意到地方就是 char[1 - 2 * !!(condition)] 这种写法，因为
!!(condition) 结果只能是0、1，所以 1 - 2 * condition) 的结果只可能是 1 或者 -1（
当 condition 为真的时候结果是 1 - 2 = -1，当 condition 为假的时候结果时 1 - 0 =
1）。这样一来，如果 condition 为真，那么最终这个宏会被扩展成
(void)sizeof(char[-1])，显然这个表达式无法通过编译，因为 char[-1] 时非法的表达式
。

这个宏定义的作用是在编译时确保某些条件成立，使得错误在编译期间就暴露出来，有点类
似与 assert() 的作用。


* * *

完
