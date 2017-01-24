---
title: Android JNI 使用总结
date: 2017-01-24 09:30:21
tags:
 - C/CPP
 - JNI
---

最近使用JNI封装项目的接口，遇到一些坑，在这里总结一下。文中提到的各种结构的定义
都是C++的定义和它们对应的C定义有些许不同。

<!--more-->

# 获取 JavaVM

`JavaVM` 是JNI定义的两大核心数据接口之一，理论上你可以为每个进程创建多个
`JavaVM`的实例，但是安卓只允许一个。获取这个实例的方式比较简单，你需要在Java代码
中像下面这样加载动态库：

```java
static {
    System.loadLibrary("your-native-lib");
}
```

在库加载的时候，下面这个函数会被调用：

```c++
jint JNI_OnLoad(JavaVM* vm, void* reserved);
```

你可以在这个函数中把参数`vm`缓存下来，因为每个进程只允许有一个`JavaVM`的实例，所
以把它当成全局变量`cache`下来应该是安全的。

# 获取 JNIEnv

JNI定义的另一个核心的数据结构是JNIEnv，这个数据结构的实例的获取方式分为两种情况
。

## 你的代码是 Java 代码的 native 方法的实现

比如如下代码：

```java
public class Widget {
private native void nativeMethod();
}
```

`nativeMethod`的实现可能是下面这个样子：

```c++
JNIEXPORT void JNICALL
Java_xxxxx_nativeMethod(JNIEnv *env, jobject instance);
```

这种情况下，JNI 会把 `JNIEnv` 当成参数传递到 native 层的 C/C++ 代码，你直接使用
就可以了。

## 如果你想要从 native 层直接调用 Java 代码

很多时候，你的native代码建立自己的线程（比如建立线程监听），并在合适的时候回调
Java 代码，我们没有办法像上面那样直接获得 `JNIEnv`，获取它的实例需要把你的线程
`Attach`到`JavaVM`上去，调用的方法是 `JavaVM::AttachCurrentThread`

```c++
JNIEnv* env;
GetJVM()->AttachCurrentThread(&env, nullptr);
```

这里你需要用到`JavaVM`的实例，这个实例的获取方式可以参考上一个小节。使用完之后你
需要调用 `JavaVM::DetachCurrentThread`函数解绑线程。

```c++
GetJVM()->DetachCurrentThread();
```

需要注意的是对于一个已经绑定到`JavaVM`上的线程调用`AttachCurrentThread`不会有任
何影响。如果你的线程已经绑定到了`JavaVM`上，你还可以通过调用`JavaVM::GetEnv`获取
`JNIEnv`，如果你的线程没有绑定，这个函数返回`JNI_EDETACHED`。我的习惯是封装一个
智能指针类自动完成这些操作。

```c++
class JNIEnvPtr {
public:
    JNIEnvPtr() : env_{nullptr}, need_detach_{false} {
        if (GetJVM()->GetEnv((void**) &env_, JNI_VERSION_1_6) ==
            JNI_EDETACHED) {
            GetJVM()->AttachCurrentThread(&env_, nullptr);
            need_detach_ = true;
        }
    }

    ~JNIEnvPtr() {
        if (need_detach_) {
            GetJVM()->DetachCurrentThread();
        }
    }

    JNIEnv* operator->() {
        return env_;
    }

private:
    JNIEnvPtr(const JNIEnvPtr&) = delete;
    JNIEnvPtr& operator=(const JNIEnvPtr&) = delete;

private:
    JNIEnv* env_;
    bool need_detach_;
};
```

这个类在构造函数中调用`AttachCurrentThread`在析构中调用`DetachCurrentThread`，然
后重载`->`操作符。你可以像下面这样使用这个工具类。

```c++
// native 代码需要回调 Java 代码

NativeClass::NativeMethod() {
    JNIEnvPtr env;
    env->CallVoidMethod(instance, method, args...);
}
```

# 获取 jclass, jmethodID, jfieldID

如果你想要调用Java层的代码，你需要使用 JNIEnv（如何获取它的实例在上一小节中已经
提到），比如你想要调用一个无返回值无参数的成员函数，语法大概如下：

```c++
env->CallVoidMethod(instance, method);
```

这里你最少需要两个参数`instance`和`method`，因为你要调用一个方法你至少需要有一
个对象以及一个成员函数名称。关于`instance`如何获取，我们放在下一个小节，这里主要
讨论如何获取`method`。

在 Java 中所有的方法必然属于某一个类，所以获取 `method` 之前你需要获取 `class`，
它们类型分别为`jclass`,`jmethod`。

## 获取类的引用

调用`JNIEnv::FindClass`可以获取对于的 class 的实例。

```c++
jclass clazz = env->FindClass("full/name/of/your/class");
```

## 获取成员函数的引用

获取一个方法的引用需要调用`JNIEnv::GetMethodID`方法。

```c++
jmethodID = env->GetMethodID(clazz, "method", "()V");
```

第一个参数是上面获取的类引用，第二个参数是方法名称，第三个参数是方法的签名。关于
签名的写法，请参考官方文档的[Type Signatures][sign]一小节

[sign]: http://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/types.html#type_signatures

需要额外注意的地方是，构造函数的函数名称为`<init>`

## 获取成员变量的引用

获取成员变量的引用和获取成员函数的引用是类似的，调用`JNIEnv::GetFieldID`即可，这
里不再赘述。

## 失效问题

特别需要注意的地方是，jclass, methodID 和 fieldID 在类 unload 之前都是有效，虽然
一个类会 unload 的情况非常少见，但是并不是没有可能。所以如果你想要保证你的这些引
用有效的话，可以通过下面这种方式对你用到的类，方法，成员变量进行缓存：

```java
private static native void nativeInit();

static {
    nativeInit();
}
```

然后在你的 native 方法中获取并缓存你用到的 entity。

```c++
Java_xxxx_nativeInit(JNIEnv *env, jobject instance) {
    // 查找并且缓存你需要用到的上述对象
}
```

当然如果你使用 `System.loadLibrary` 你也可以在 `JNI_OnLoad` 函数中缓存这些东西。

## 辅助工具

下面这两个宏定义来自 `libvlc-android` 可以用来方便的获取 jclass，jmetod，jfield

```c++
#define GET_CLASS(clazz, str, b_globlal)                                    \
    do {                                                                    \
        (clazz) = env->FindClass((str));                                    \
        if (!(clazz)) {                                                     \
            return -1;                                                      \
        }                                                                   \
        if (b_globlal) {                                                    \
            (clazz) = reinterpret_cast<jclass>(env->NewGlobalRef((clazz))); \
            if (!(clazz)) {                                                 \
                return -1;                                                  \
            }                                                               \
        }                                                                   \
    } while (0)

#define GET_ID(get, id, clazz, str, args)                 \
    do {                                                  \
        (id) = env->get((clazz), (str), (args));          \
        if (!(id)) {                                      \
            return -1;                                    \
        }                                                 \
    } while (0)
```

使用方式如下：

```c++
GET_CLASS(clazz, "full/class/name", false); // 第三个参数后面解释
GET_ID(GetMethodID, method, clazz, "method", "()V");
GET_ID(GetFieldID, field, clazz, "filed", "I");
```

此外这些 class, methodID， filedID 是不会变的东西（从逻辑上理解，类名，方法名，
成员名都不会在运行是更改），你可以缓存你查找好的引用作为全局变量，一方面可以提升
效率，另一方面也方便使用，因为你不需要每次都重新查找。个人的习惯是写一个结构体的
单例，在 `JNI_OnLoad`中查找并缓存这些实体，然后在`JNI_OnUnload`中清理缓存。

# 获取 instance

前面提到，如果想要在 native 层调用 java 层的函数，你至少一个对象和一个成员方法，
上一小节讲述了如何获取成员方法，这一小节主要讲述如获取一个Java对象也就是
instance。获取的它通常是通过一下三种途径：

## native 代码参数

`instance`的获取和`JNIEnv`的获取是类似的，如果你的方法是Java代码的native实现，那
么JNI会自动把调用该对象的实例传递给你。

```c++
JNIEXPORT void JNICALL
Java_xxxxx_nativeMethod(JNIEnv *env, jobject instance);
```

上面函数的第二个参数就是你需要的对象。你可以直接在这个函数中使用这个对象，比如：

```c++
JNIEXPORT void JNICALL
Java_xxxxx_nativeMethod(JNIEnv *env, jobject instance) {
    auto& cached_fields = CachedFields::GetInstance();  // 参考上一小节
    env->CallVoidMethod(instance, cached_fields.methodID)
}
```

## CallObjectMethod

在`instance`存在的情况下，你也可以调用这个`instance`返回对象的方法来获取另一个
`instance`，比如：

```c++
jobject obj_ = env->CallObjectMethod(instance, methodID, args);
```

## NewObject

如果你没有这样的 instance 存在，你也可以直接创建一个java对象，调用
`JNIEnv::NewObject`即可。

```
jobject obj_ = env->NewObject(fields.clazz, fields.ctrID);
```

这个函数需要 jclass 和构造函数的 jmethod（关于如何获取构造函数的引用可以查看上一
小节）。

# 引用的局部性和全局性

所有传递到native函数中的参数和从JNI函数中返回的对象都是局部引用，比如：

```c++
JNIEXPORT void JNICALL
Java_xxxxx_nativeMethod(JNIEnv *env, jobject instance) {
    jobject obj_ = env->CallObjectMethod(instance, methodID, args);
}
```

`instance` 和 `obj_` 都是局部引用，这种引用一旦函数返回就会失效，即使你保存它们
也不会延长它们的生命周期。

```c++

jobject instance_backup;

JNIEXPORT void JNICALL
Java_xxxxx_nativeMethod(JNIEnv *env, jobject instance) {
    instance_backup = instance;
}

NativeClass::NativeMethod() {
    JNIEnvPtr env;
    env->CallVoidMethod(instance_backup, methodID); // 错误！！！！！
}
```

这条规则对于所有的 jobject 的子类（包括 jclas， jstring，jarray）都是适用的，如
果你想要引用保持有效，你需要调用`JNIEnv::NewGlobalRef`来获取一个全局的引用。

```
jclass localClass = env->FindClass("MyClass");
jclass globalClass = reinterpret_cast<jclass>(env->NewGlobalRef(localClass));
```

在你调用 DeleteGlobalRef 之前，这个引用都会有效。所以如果你想要保存一个局部引用
，你可以像下面这样组织你的代码：

```
class Widget {
public:
    Widget(jobject instance) {
        JNIEnvPtr env;
        instance_ = env->NewGlobalRef(instance);
    }

    ~Widget() {
        JNIEnvPtr env;
        env->DeleteGlobalRef(instance_);
    }

    void Function() {
        // 使用 instance_;
    }

private:
    jobject instance_;
};
```

# 全局和局部引用的删除

全局引用需要你手动调用 `DeleteGlobalRef` 来删除，但是局部的引用通常不需要。但是
需要注意的是，如果你使用了 AttachCurrentThread 绑定线程，那么在你调用
DetachCurrentThread 之前，你的局部引用都不会自动回收，这意味着如果你在一个循环中
创建了局部引用，你通常需要在循环内部删除掉它，因为系统通常只保证了 16 个局部引用
的 slot 。如果你需要超过 16 个，你就必须删除一些local引用，或者使用
EnsureLocalCapacity/PushLocalFrame 为局部引用预留更多的slot。

# 成对的使用函数

JNI中有一些函数是需要成对的使用的，否则会有内存泄露，常用的有以下这些。

```
const char * GetStringUTFChars(jstring string, jboolean *isCopy);
void ReleaseStringUTFChars(jstring string, const char *utf);

NativeType *Get<PrimitiveType>ArrayElements(ArrayType array, jboolean *isCopy);
void Release<PrimitiveType>ArrayElements(ArrayType array, NativeType *elems, jint mode);
```

前面这一组用来操作字符串，后面这一组用来操作数组。

# 参考

本文中的内容主要参考：

1. [JNI Tips](https://developer.android.com/training/articles/perf-jni.html)
2. [Java Native Interface Specification Contents](http://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/jniTOC.html)
