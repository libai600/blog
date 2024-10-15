---
title: 'C++11：static_assert 静态断言'
slug: cpp11-static-assert
date: 2016-08-22
tags: [c++]
---

C++11 引入了 `static_assert`，用来做编译期间的断言，因此叫做静态断言。

## 静态断言

静态断言使用非常简单，语法如下：

```c++
static_assert(常量表达式，错误字符串)
```

注意，因为是编译期的检查，这里的表达式必须是常量表达式，也就是能够在编译时计算出来的值，不能是变量。错误字符串也必须是字面量，不能是字符串变量。

当 `static_assert` 中的表达式计算结果为 false 时，编译器就会报出错误，并显示错误字符串的信息。例如：

```c++
//检查编译环境是否为64位
static_assert(sizeof(void*) == 8, "only 64bit is supported.");
```

如果使用 32 位编译器编译这个代码，编译器就会报错，比如：

```
error: static assertion failed: only 64bit is supported.
```

## static_assert 与 assert

它们都是断言，通常用于检查发现程序内部的一些错误，程序外部因素引发的异常（如用户输入错误）不应该由断言处理。

- `assert` 是运行时断言，只能在程序运行的时候才能发现问题。`static_assert` 是静态断言，在编译期间检查程序问题。

- `assert` 通常用于检查变量，`static_assert` 只能检查常量表达式。

- `assert` 只能作为普通的代码语句出现在函数实现中。`static_assert` 没有使用范围限制，可以在函数外使用。

- `assert` 有运行时的消耗，对程序性能有影响，通常在 release build 中被 `NDEBUG` 宏控制关闭。`static_assert` 是编译期的检查，没有运行时消耗，不影响程序性能，在 release build 中仍然有效。
