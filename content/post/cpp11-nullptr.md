---
title: 'C++11：nullptr 空指针'
slug: cpp11-nullptr
date: 2016-07-20
tags: [C++]
---

C++11 引入了一个新的关键字 `nullptr` 来表示空指针。

在 C++11 之前，表示空指针一般有两种方式：`0` 或着 `NULL`。但不同编译器对 `NULL` 的解释可能并不一致。这种表示空指针的方式一般都能正常的表达我们的逻辑，但在某些情况下可能会引起一些问题。比如下面的代码：

```c++
void print(int i) {
    printf("number: %d\n", i);
}
void print(void* c) {
    printf("address: %p\n", c);
}

int main()
{
    int i = 32;
    char* c = 0;
    print(i);    // 调用print(int)
    print(c);    // 调用print(void*)
    print(NULL); // 调用哪个print？
}
```

在 `print(NULL)` 这句中，程序员的本意显然是想要打印空指针的值（虽然这么写没什么实际作用），但结果可能并不如预期。在 MSVC 中，`NULL` 是一个值为 0 的宏定义，所以编译器调用的其实是 `print(int)` 的版本，并且也没有任何警告信息。在 GCC 中，`NULL` 也是一个宏，但它的值是一个内部标识 `__null`，`print(NULL)` 会导致 GCC 报错，错误为“不明确的重载调用”。

为了解决这种因为0既表示空指针也表示数字引起的问题，C++11 引入了一个新的关键字 `nullptr`，用于专门表示空指针。它可以隐式的转换为任意类型的指针类型，但不能转为非指针的类型，并且 `nullptr` 只能与指针类型的数据进行比较。

```c++
char* c = nullptr;    // OK
if (c) { }            // OK
if (c == nullptr) { } // OK

int i = nullptr;      // Error
```

如果把前面例子中的 `print(NULL)` 改为 `print(nullptr)`，就能避免刚才的问题，因为 `nullptr` 可以被转为 `void*`，但不能被转为 `int` 类型。

也可以认为 `nullptr` 就是一个新版的 `NULL`，但不会像 `NULL` 一样可能导致潜在的问题，以前用 `NULL` 或者 `0` 表示空指针的地方，现在也可以使用 `nullptr` 来表示。

