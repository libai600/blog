---
title: 'C++11：原始字符串字面量'
slug: cpp11-raw-string-literals
date: 2016-09-29
tags: [C++]
---

C++11 引入了一种表示原始字符串字面量的语法，简单来说就是所见即所得，写的什么就是什么。

在 C/C++ 中，字符串有很多转义字符，比如 `\r\n\t`，它们并不是字面上的含义，而是具有特殊的表示。C++11 引入了一种表示原始字符串字面量的语法，简单来说就是所见即所得，写的什么就是什么。来看下面的例子：

```c++
const char path1[] = "C:\\path\\code\\example"; // C++98
```

在 C++98 中，如果我们想表示字符串中的反斜杠 `\`，必须用 `\\`表示。如果想表示 Windows 系统路径或者正则表达式，会比较不方便。C++11 的引入的原始字面量语法解决了这一问题，原始字面量以 R 开头，然后使用一对双引号和小括号包含。例如：

```c++
// C++11原始字面量
const char path1[] = R"(C:\path1\code\example)"; 
const char path2[] = R"(D:\path2\code\example)"; 
const char str1[] = R"(abc\ndef)"; // 这里的 \n 并不会换行 
const char str1[] = R"(opq\trst)"; // 这里的 \t 也不会缩进
std::cout << path1 << std::endl
          << path2 << std::endl
          << str1  << std::endl
          << str2  << std::endl;
```

这个例子的输出为：

```
C:\path1\code\example
D:\path2\code\example
abc\ndef
opq\trst
```

在双引号和小括号之间还可以加入一些字符作为分隔符（不能为括号，反斜杠或空格），但两端的分隔符必须保持一致，这些分隔符并不属于字符串的内容。例如：

```c++
const char path1[] = R"aa(C:\path1\code\example)aa"; 
const char path2[] = R"###(D:\path2\code\example)###"; 
```