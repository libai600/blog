---
title: 'C++11：noexcept'
slug: cpp11-noexcep
date: 2016-08-12
tags: [C++]
---

在 C++ 中，有时为了防止异常扩散，或者方便编译器优化，我们会明确表示某个函数不会抛出异常。在 C++11 之前，需要在函数声明后面加上 `throw()` 来表示函数不会抛出异常。C++11 改进了这项设计，引入了 `noexcept`，它既是一个修饰符，也是一个运算符。

## noexcept修饰符

将 `noexcept` 关键字放在函数声明和定义的后面，表示这个函数不会抛出异常。例如：

```c++
void foo() noexcept; // C++11的写法
void doo() throw();  // C++98的写法
```

这种 `noexcept` 修饰并不是“强制的”，如果在 `foo` 函数中使用了 `throw` 语句，程序仍然能编译通过，不过很可能产生一些警告。在运行时，如果在 `foo` 中真的抛出了异常，程序会直接调用 `std::terminate()` 中断运行。这一点和使用 `throw()` 来修饰函数的默认效果是相同的，但机制其实略有不同，`noexcept` 的效率会好一些。

`noexcpt` 修饰符后面还可以接受一个常量表达式。当表达式值为 true 时，和单独的 `noexcept` 相同，表示函数不会抛出异常，当表达式值为 false 时，表示函数可能抛出异常。例如：

```c++
void foo noexcept(true); //不会抛出异常
void doo noexcept(false);//可能抛出异常
```

这种用法常用在泛型编程中，例如，

```c++
template <typename T>
void foo() noexcept(std::is_pod<T>::value) {
}
```

如果 T 是 POD 类型，函数不会抛出异常，否则可能抛出异常。

## noexcept运算符

`noexcept` 也可以作为运算符，用来检测某个函数是否会抛出异常，如果函数被声明为不会抛出异常，则结果为 true，否则结果为 false。例如：

```c++
void no_throw() noexcept {  }
void may_throw() { }

template <typename T>
void foo() noexcept(std::is_pod<T>::value)
{
}

template <typename T>
void doo(T value) noexcept(std::is_pod<T>::value)
{
    std::cout << value << std::endl;
}

void main()
{
    std::cout << std::boolalpha
              << noexcept(no_throw()) << std::endl  // true
              << noexcept(may_throw()) << std::endl // false
              << noexcept(foo<int>()) << std::endl  // true
              << noexcept(foo<std::string>()) << std::endl // false
              << noexcept(doo(1)) << std::endl  // true
              << noexcept(doo(std::string("123"))) << std::endl;  // false
}
```

运行输出为：

```c++
true
false
true
false
true
false
```

需要注意，`noexcept` 作为运算符时，这种检查和计算结果是在编译时刻确定的，不会在运行时执行 `noexcept` 参数表达式。例如上例中的 `noexcept(doo(1))` 和 `noexcept(doo(std::string("123")))`，在程序运行时不会执行 `doo` 函数，也没有相应的打印输出。

`noexcept` 作为修饰符和运算符，可以结合起来使用，例如：

```c++
void foo() noexcept(noexcept(doo()))
{
    doo();
}
```

这个例子表示，函数 `foo` 是否会抛出异常与函数 `doo` 有关，如果 `doo` 不会抛出异常，`foo` 就不会抛出异常，如果 `doo` 可能抛出异常，`foo` 也可能抛出异常。这种用法也非常适合应用在泛型编程上。例如：

```c++
template <typename T>
void doo(T value) noexcept(std::is_pod<T>::value)
{
    std::cout << value << std::endl;
}

template <typename T>
void foo(T value) noexcept(noexcept(doo(value)))
{
    doo(value);
}
```


所以通过 `noexcept`，我们可以更方便的设计异常的处理。