---
title: Windows 上动态库符号的导出和导入
slug: dll-exporting-importing
date: 2015-01-31
tags: [Windows]
description: 
---

在 Windows 系统上，如果使用 MSVC 的开发环境，创建和链接动态库时，需要设置符号的导出和导入。

动态库中，如果要将某个类或函数提供给外部使用，需要导出相应的符号，使用者要用这些函数的话，也需要导入对应的符号。如果没有正确的导入导出，在链接动态库时就会报出 "error LNK2019: 无法解析的外部符号" 的错误。

通常有两种方式处理 DLL 的导出导入问题，一种是使用额外的 `.def` 文件声明的方式，另一种是在代码中使用 `__declspec` 声明的方式，这里我们介绍后一种方法。

导出符号使用 `__declspec(dllexport)` 关键字，放在函数声明的前面，或者类名前面。例如，

```c++
__declspec(dllexport) void foo();
class __declspec(dllexport) SomeData
{
    ......
};
```

导入符号使用 `__declspec(dllimport)` 关键字，使用位置和导出一样。例如，

```c++
__declspec(dllexport) void foo();
class __declspec(dllexport) SomeData
{
    ......
};
```

动态库的开发者需要提供一个接口头文件给使用者，为了方便，这份头文件当然最好就是开发者自己使用的那个。对于提供者，需要的是导出，对于使用者，需要的是导入，同一份头文件需要既用于导出又用于导入，可以借助一些宏定义来实现。例如，

```c++
#ifndef FOO_H
#define FOO_H

#ifdef BUILD_FOO_LIB
  #define FOO_API __declspec(dllexport)
#else
  #define FOO_API __declspec(dllimport)
#endif

FOO_API void foo();

class FOO_API SomeData
{
public:
    SomeData() {}
};

#endif // FOO_H
```

库的开发者编译产生库时，需要添加 `BUILD_FOO_LIB` 的宏定义给编译器，这样，对于提供者来说 `FOO_API` 就是导出，而使用者不会定义 `BUILD_FOO_LIB`，所以 `FOO_AP` 就是导入。

参考：

- https://docs.microsoft.com/en-us/cpp/build/exporting-from-a-dll-using-declspec-dllexport
- https://docs.microsoft.com/en-us/cpp/build/importing-into-an-application-using-declspec-dllimport
