---
title: GCC 的 -fvisibility 参数
slug: gcc-fvisibility
date: 2015-02-28
tags: [linux]
---

在 Windows 上使用 MSVC 创建和使用 DLL 动态库时需要导出导入符号（ 参见[Windows上动态库符号的导出和导入]({{< ref "/post/dll-exporting-importing" >}}) ），但在 Linux 上使用 GCC 时，一般好像不需要导入导出符号。其实不然，GCC 编译时并不是不需要导出符号，而是默认导出了所有的符号。

GCC 中也存在一个符号可见性的概念，称之为 Visibility，一般指的就是动态库中符号的可见性。默认情况下，.so 动态库中的所有符号对于外部都是可见的，因此使用者可以直接使用动态库提供的函数。但 GCC 提供了修改可见性的方式，编译时可以通过 `-fvisibility` 参数修改默认所有符号的可见性，也可以在代码中使用 `__attribute__` 来修改某个符号的可见性。

那既然 GCC 默认已经让所有符号都可见，为什么还要提供选项来隐藏符号呢？符号全部可见岂不是更方便，不用像在 Windows 上那样麻烦的导出导入。其实不是，GCC 推荐隐藏动态库内部使用的符号，只把需要提供给外部使用的符号设置为可见。GCC 的解释为：通过隐藏那些不需要外部使用的符号，可以减少动态库的加载时间，减小动态库文件的大小，能够让编译器生成更优的代码，同时也能避免不同库之间的符号冲突。所以我们在设计动态库时，可以先通过 `-fvisibility` 参数使所有符号默认不可见，再通过 `__attribute__` 使那些需要提供给外部的符号变为可见。

## -fvisibility

`-fvisibility` 参数的完整形式如下：

```
-fvisibility=[default|internal|hidden|protected]
```

visibility 有四个可以设置的值，其中 internal 和 protected 很少使用到，大部分情况下只需要使用 default 和 hidden 两个值。default 就是 GCC 的默认情况，表示所有符号都是可见的，hidden 表示设置所有符号都是不可见的。

编译动态库时，添加 `-fvisibility=hidden` 参数，使所有符号都默认不可见，例如，

```
g++ -fvisibility=hidden -c foo.cpp -o foo.o
```

## __attribute__((visibility("default")))

`__attribute__` 中的 visibility 属性用于单独修改某个符号的可见性，会覆盖 `-fvisibility` 参数的设置。我们在头文件中声明时，将需要提供给外部使用的符号设为可见，例如，

```c++
__attribute__((visibility("default"))) void foo();

class __attribute__((visibility("default"))) Foo
{
    ...
};
```

对于库的提供者，编译时需要使用 `__attribute__((visibility("default")))` 来将符号设为可见，对于库的使用者，这个关键字是可选的，不必像 Windows 上那样一个导出一个导入，但我们可以使用宏来替换这个比较长的关键字，并且兼容 Windows 上的机制。

```c++
#ifndef FOO_H
#define FOO_H

#ifdef _WIN32
  #define FOO_EXPORT __declspec(dllexport)
  #define FOO_IMPORT __declspec(dllimport)
#else
  #define FOO_EXPORT __attribute__((visibility("default")))
  #define FOO_IMPORT
#endif

#ifdef FOO_DLL
  #define FOO_API FOO_EXPORT
#else
  #define FOO_API FOO_IMPORT
#endif

FOO_API void func();

class FOO_API Data
{
    ...
};

#endif
```

参考：

- https://gcc.gnu.org/wiki/Visibility
- https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html
