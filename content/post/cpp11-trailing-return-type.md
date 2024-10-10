---
title: 'C++11：后置返回类型'
slug: cpp11-trailing-return-type
date: 2016-07-03
tags: [c++]
---

在 C++ 中，我们声明函数时，都是把返回值类型写在前面，比如，

```c++
double foo(int i); // 返回值类型为double
```

然而在 C++11 中，引入了一种新的函数声明语法，可以将返回值类型后置：

```c++
auto foo(int i) -> double;
```

这里的 `-> double` 就表示函数返回值类型为 `double`，而原本放置返回值类型的地方需要使用 `auto` 替代。

通常情况下这种新的写法并没有什么优点，甚至比传统写法更麻烦，但在泛型编程场景中结合 `decltype` 就会体现出它的方便之处。来看这个例子：

```c++
template <typename R, typename T1, typename T2>
R sum(T1 a, T2 b)
{
    return a + b;
}

int main()
{
    int n = 1;
    double m = 2.0;
    auto s = sum<double>(n, m);
    return 0;
}
```

由于函数 `sum` 的返回值类型在被调用时才能确定，我们只能添加一个用于表示返回值类型的模板参数 `R`，并在调用 `sum` 时指定这个类型，所以即便我们使用了 `auto` 来声明 `s`，程序员还是需要明确知道这种情况下 `sum` 的返回类型，感觉起来和“自动类型”还是差了点意思。

而如果结合后置返回类型和 `decltype`，就能实现真正优雅的自动类型推导：

```c++
template <typename T1, typename T2>
auto sum(T1 a, T2 b) -> decltype(a + b)
{
    return a + b;
}

int main()
{
    int n = 1;
    double m = 2.0;
    auto s = sum(n, m);
    return 0;
}
```

这个新的 `sum` 函数模板使用了后置返回类型的声明，并且返回类型使用 `decltype(a + b)` 来进行推导，这样在调用 `sum` 时就很简洁直观了。这种将返回类型后置和 `decltype` 结合的方式，也成为追踪返回类型。

可能有人会问，为什么不直接像下面这样写，岂不是更加简单：

```c++
template <typename T1, typename T2>
decltype(a + b) sum(T1 a, T2 b)
{
    return a + b;
}
```

确实更加直观，但实际上行不通，编译器会报错，原因是编译器在尝试对 `decltype(a + b)` 进行推导时，`a` 和 `b` 还未声明，尽管它们就在后面，你可以理解为编译器只能从左到右进行处理。这也是为什么引入一种新的语法，将返回值后置的主要原因。

返回类型后置和 `decltype` 结合还经常用在转发函数中。例如，

```c++
int& foo(int& i);
float foo(float& f);

template <typename T>
auto forward(T& val) -> decltype(foo(val))
{
    return foo(val);
}
```

这种统一的转发函数在 C++11 之前是无法实现的。

---

参考：

- https://en.cppreference.com/w/cpp/language/function
