---
title: Visual Studio 各命令行提示符的区别
slug: vs-prompt
date: 2015-01-02
tags: [Windows]
description: 
---

在 Windows 上，构建一些代码项目的时候，可能需要在命令行下使用 MSVC 的编译环境，这时候就需要用到 Visual Studio 提供的命令提示符工具（Command Prompt），这些工具可以在开始菜单的 VS 目录下找到。

但 VS 一般都提供了好几种命令提示符，例如 VS2015 版就提供了6种，乍一看一头雾水，完全不知道该用哪一个。

| 中文版名字 | 英文版名字 | 
| --------  | -------- |
| x86 本机工具命令提示符 | x86 Native Tools Command Prompt |
| x86 x64 兼容工具命令提示符 | x86 x64 Cross Tools Command Prompt |
| x86 ARM 兼容工具命令提示符 | x86 ARM Cross Tools Command Prompt |
| x64 本机工具命令提示符 | x64 Native Tools Command Prompt |
| x64 x86 兼容工具命令提示符 | x64 x86 Cross Tools Command Prompt |
| x64 ARM 兼容工具命令提示符 | x64 ARM Cross Tools Command Prompt |

找到这些命令提示符的位置，右键查看快捷方式的属性就可以发现，其实它们都是调用的同一个批处理文件 vcvarsall.bat，只不过后面传递的参数不同。比如 "x64 本机工具命令提示符" 的参数为 amd64，而 "x64 x86 兼容工具命令提示符" 的参数为 amd64_x86。

| 提示符 | vcvarsall.bat 参数 | 
| --------  | -------- |
| x86 本机工具命令提示符 | x86 |
| x86 x64 兼容工具命令提示符 | x86_amd64 |
| x86 ARM 兼容工具命令提示符 | x86_arm |
| x64 本机工具命令提示符 | amd64 |
| x64 x86 兼容工具命令提示符 | amd64_x86 |
| x64 ARM 兼容工具命令提示符 | amd64_arm |

可以根据参数理解它们的含义，x86，amd64，arm都是指的处理器架构，一般可以理解为 x86 就是通常说的 32位程序，amd64 就是 64 位程序，arm 就是 arm 平台上的程序。

有关处理器架构名称，可以参考：[简单区分 x86, x64, i386, AMD64, Intel64, IA-32, IA-32e]({{< ref "/post/system-arch.md" >}})

而这些参数的意思是：

- x86：编译器本身是 x86 的，产生的程序也是 x86 的。
- x86_amd64：编译器本身是 x86 的，产生的程序是 x64 的。
- x86_arm：编译器本身是 x86 的，产生的程序是 arm 的。
- amd64：编译器本身是 x64 的，产生的程序是 x64 的。
- amd64_x86：编译器本身是 x64 的，产生的程序是 x86 的。
- amd64_arm：编译器本身是 x64 的，产生的程序是 arm 的。


再对照参数和提示符名字，就能知道在什么情况下使用哪个提示符了。


*参考： https://docs.microsoft.com/en-us/cpp/build/building-on-the-command-line*