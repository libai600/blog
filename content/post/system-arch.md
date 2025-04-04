---
title: 简单区分 x86, x64, i386, AMD64, Intel64, IA-32, IA-32e
slug: x86-x64
date: 2014-12-01
description: 
---

常见的 CPU 指令集架构有 Intel 和 AMD 的 x86 架构，IBM 的 PowerPC 架构，以及 ARM 架构。而 x86 架构下又有各种不同的类型区分，比如 x86-64，AMD64，i386，Intel64，IA-32，IA-32e 等。有时候，在下载一些 Linux 系统或者选择一些编译器的时候，经常会出现这些不同的类型让你选择，容易搞混。

x86 泛指一系列基于 Intel 8086 处理器且向后兼容的 CPU 指令集架构，其最早实现在 Intel 推出的 8086 处理器上，是一款 16 位处理器。又因为 Intel 后续几款处理器的名字都以 "86" 作为结尾，比如 80186，80286，80386 等，所以该指令集架构被称为 x86 架构。

后来，x86 指令集架构发展出了两个最常见的版本，32 版和 64 位版。

32 位版的 x86 架构，最早实现在 Intel 推出的 80386 处理器上，此架构被称为 IA-32 ( Intel Architecture 32bit )，也叫 i386。

64 位版的 x86 架构，最早由 AMD 率先设计实现，并称其为 AMD64。后来 Intel 也实现了这个指令集，但 Intel 显然不想用 AMD64 这个名, 所以另外起名，早期称之为 IA-32e 或 EM64T，后来称为 Intel64。

再后来，大家为了方便理解避免混乱，将 64 位版的 x86 架构称为 x86-64，有时干脆简称 x64。相对应的，将 32 位版的 x86 架构称为 x86-32，有时和 x64 同时出现时，也直接简称为 x86。

所以，通常来说：

- x86 32bit = x86-32 = IA-32 = i386
- x86 64bit = x86-64 = x64 = AMD64 = Intel64 = IA-32e = EM64T

以上的总结中忽略了一些细微差别，比如 AMD64 和 Intel64，它们确实存在一些细节的不同，但对于普通的开发和运维人员来说，绝大多数情况下可以认为它们是相同的指令集架构。

另外，有时还会碰到另一个名字：IA-64。这个和前面说的都不同。在和 AMD 的竞争过程中，Intel 曾推出过一个 64 位的指令集，叫做 IA-64，应用在安腾（Itanium）处理器上。但是 IA64 并不属于 x86 架构家族，不能像 x86-64 一样兼容 x86-32，再加上其他一些原因，现在已基本被市场抛弃。

参考：

- https://en.wikipedia.org/wiki/X86
- https://en.wikipedia.org/wiki/IA-32
- https://en.wikipedia.org/wiki/X86-64