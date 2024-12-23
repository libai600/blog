---
title: 'C++11：模板的改进'
slug: cpp11-template-improvement
date: 2016-09-26
tags: [C++]
---

C++11 对模板进行了一些改进。

## 右尖括号的改进

在 C++98 中，使用模板经常会遇到一个头疼的问题，就是右尖括号的处理。因为连续两个右尖括号 `>>` 会被编译器解释成右移操作符，而不是模板参数的结尾，所以不得添加额外的空格将它们隔开。比如：

```c++
// in C++98
std::map<int, std::vector<int>> data; // error
std::map<int, std::vector<int> > data;// ok
```

这种设计经常会导致程序员一不留神就写错，因此在 C++11 中取消了这种限制，新标准要求编译器根据上下文判断 `>>` 到底是右移操作还是模板参数的结尾，所以不再需要额外的空格。不过，由于 C++98 老标准中的这项限制十分反人类，很多编译器在 C++11 颁布之前很早就已经支持了智能判断，因此在之前你可能并没有意识到有什么不方便之处。

## 模板的别名

在 C++11 中，可以通过 `typedef` 来定义一个类型的别名。比如：

```c++
typedef unsigned char byte;
```

就定义了一个 `unsigned char` 类型的别名，称为 `byte`。但需要注意的是，这里的 `byte` 并非是一个新的类型，它只是 `unsigned char` 的一个别称。

当遇到比较长的类型名字，比如模板类的时候，这种定义别名的方法就会十分方便。比如：

```c++
typedef std::vector<std::string> string_vector;
typedef std::map<int, std::string> data_map;
```

但 `typedef` 也存在一定的局限性。当我们想要为一个key为 `int` 类型，value为泛化类型 `T` 的 `std::map` 取别名时，`typedef` 就无能为力了。

```c++
typedef std::map<int, T> id2T; // 并不允许这么写
```

现在，C++11 引入了一种新的定义别名的方式 `using`，就可以支持这种情况。

```c++
template <typename T> using id_map = std::map<int, T>;
// id_map<T> 等价于 std::map<int, T>
```

此处使用新的 `using` 语法定义了一个 `std::map<int, T>` 类型的别名 `id_map`，因此，`id_map<T>` 就等价于 `std::map<int, T>`，从而就可以利用这个别名来定义不同的 `map`，简化代码的编写。例如：

```c++
#include <map>
#include <string>

template <typename T> 
using id_map = std::map<int, T>;

int main()
{
    id_map<std::string> id2name;
    id_map<float> id2price;
    return 0;
}
```

`using` 不仅可以用来定义模板的别名，它也可以用来定义其他类型的别名，完全可以覆盖 `typedef` 的所有功能。例如下面这些 `typedef` 和 `using` 的写法就是等价的：

```c++
typedef unsigned char byte;
using byte = unsigned char;

typedef std::vector<std::string> string_vector;
using string_vector = std::vector<std::string>;
```

在定义函数指针类型时，使用 `using` 会更加直观易读：

```c++
typedef void(*func_t)(int, int);
using func_t = void(*)(int, int);
```

而前面提到的 `using` 定义模板别名的语法，只是在普通类型别名语法的基础上增加了 `template` 的参数列表，就可以定义一个模板的别名。

## 函数模板的默认模板参数

在 C++98 中，类模板允许有默认模板参数，而函数模板却不允许有默认模板参数。例如：

```c++
template <typename T = int>
class MyClass { };

template <typename T = int>
void my_func(); // C++98不允许
```

但在最新的 C++11 中，取消了这项限制，函数模板也可以有默认的模板函数。并且它和其他默认参数的语法略有不同，它不要求默认参数必须从右到左写在参数列表的最后面。比如这样写也是完全可以的：

```c++
template <typename T1 = int, typename T2>
void my_func();
```

函数模板的参数推导规则也并不复杂，简单来说，如果能够从函数实参中推导出类型，那么默认的模板参数就不会被使用，反之，默认模板参数则可能会被使用。比如下面这个例子：

```c++
template <typename T, typename U = double>
void f(T t = 0, U u = 0);

void g()
{
    f(1, 'c');      // f<int, char>(1, 'c') 
    f(1);           // f<int, double>(1, 0) 使用了默认模板参数double
    f();            // 错误，T无法被推导出来
    f<int>();       // f<int, double>(0, 0) 使用了默认模板参数double
    f<int, char>(); // f<int, char>(0, 0)
}
```

有一点需要注意，模板函数的默认形参不是模板参数推导的依据，模板参数的选择总是由函数的实参推导而来。
