---
title: 'C++11：functional'
slug: cpp11-stl-functional
date: 2016-10-15
tags: [C++]
---

## 可调用对象

在 C++ 中，有一个**可调用对象（Callable）**的类型概念，大概包括以下这么几种常见的类型：


1. 函数指针
2. 仿函数
3. Lambda 函数
4. 可被转换为函数指针的对象
5. 类成员函数指针


顾名思义，这些可调用对象的共同点就是能够被调用：

```c++
#include <iostream>

void print(const char* s)
{
    std::cout << s << std::endl;
}

struct Foo
{
    void operator()(const char* s)
    {
        std::cout << s << std::endl;
    }
};

struct Bar
{
    void my_print(const char* s)
    {
        std::cout << s << std::endl;
    }
};

int main()
{
    // 函数
    print("function");

    // 函数指针
    void (*f_ptr)(const char*) = &print;
    f_ptr("function pointer");

    // 仿函数
    Foo foo;
    foo("functor");

    // Lambda函数
    auto lambda = [](const char* s){ std::cout << s << std::endl; };
    lambda("lambda function");

    // 类成员函数指针
    void (Bar::*mem_func_ptr)(const char*) = &Bar::my_print;
    Bar bar;
    (bar.*mem_func_ptr)("member function pointer");

    return 0;
}
```

虽然这些可调用对象有比较统一的操作形式（加括号表示进行调用），但它们的定义方式却是五花八门，没有较为统一的形式进行表示和保存。这就导致当我们试图用统一的方式传递一个可调用对象时，就会发现非常麻烦难以实现。

C++11 扩展了 `functional` 库，主要是添加了 `std::function` 和 `std::bind`，用来可调用对象的各种操作。

## std::function

`std::function` 是一个可调用对象包装器，它是一个类模板，可以容纳除了类成员函数指针之外的所有可调用对象，它可以用统一的方式处理函数、函数对象、函数指针等，并允许保存和延迟执行它们。

来看下面这个例子：

```c++
#include <iostream>
#include <functional>

void print(const char* s)
{
    std::cout << s << std::endl;
}

struct Foo
{
    void operator()(const char* s)
    {
        std::cout << s << std::endl;
    }
};

int main()
{
    std::function<void(const char*)> ff = print;
    ff("function");

    void (*f_ptr)(const char*) = &print;
    ff = f_ptr;
    ff("function pointer");

    Foo foo;
    ff = foo;
    ff("functor");

    auto lambda = [](const char* s){ std::cout << s << std::endl; };
    ff = lambda;
    ff("lambda function");

    return 0;
}
```

在这个例子中，`std::function<void(const char*)>` 中的模板参数 `void(const char*)` 就是可调用对象的类型签名（也就是通常所说的函数签名）。当我们给 `std::function` 填入对应类型签名之后，它就变成了一个可表示所有这一类可调用类型的包装器。同时，在这个例子中也可以看到，`std::function` 还可以通过赋值，来改变它所包装的可调用对象。

因此，`std::function` 完全可以取代传统的函数指针，因为它可以保存包括函数指针在内的可调用对象，用于延迟执行，所以很适合作为回调函数的表示方式。例如：

```c++
class Notifier
{
    std::function<void(int)> _callback;

public:
    void set_callback(const std::function<void(int)> &func)
    {
        _callback = func;
    }

    void notify(int n)
    {
        _callback(n);
    }
};

void foo(int n) { printf("foo: %d\n", n); }

int main()
{
    Notifier nn;

    nn.set_callback(&foo);
    nn.notify(123);

    nn.set_callback([](int n){ printf("lambda: %d\n", n); });
    nn.notify(456);

    return 0;
}
```

需要注意的是，`std::function` 定义了默认的构造函数，可以构造一个空的 `std::function` 对象，此时它没有包装任何的可调用对象，如果对其进行调用操作，就会抛出一个 `std::bad_function_call` 类型的异常。比如，在上面的例子中，假如我们不设置一个回调函数而直接调用 `notify`，就会导致运行时的异常产生：

```c++
Notifier nn;
nn.notify(123); // 运行时抛出std::bad_function_call异常
```

`std::function` 也可以用来包装成员函数，但和包装普通的函数稍有差别，参考下面这个例子：

```c++
#include <stdio.h>
#include <functional>

class Foo
{
    int _n;
public:
    static void print_name() { printf("%s\n", "FOO"); }

    Foo(int num) : _n(num){ }

    void print() const
    {
        printf("%d\n", _n);
    }

    void add(int i)
    {
        _n += i;
        printf("%d\n", _n);
    }
};

int main()
{
    // 静态成员函数的包装和普通函数相同
    std::function<void()> func = &Foo::print_name;
    func();

    Foo foo(0);

    std::function<void(Foo&)> func1 = &Foo::print;
    func1(foo); // == foo.print()

    std::function<void(Foo&, int)> func2 = &Foo::add;
    func2(foo, 1); // == foo.add(1)

    return 0;
}
```

使用 `std::function` 包装静态成员函数时，与包装普通的函数相同，在这里的 `&Foo::print_name` 中，`Foo::` 其实就相当于一个 namespace。

使用 `std::function` 包装非静态成员函数时，情况有了一些变化，用来表示可调用类型的模板参数中，第一项参数必须为对应类型的引用：

```c++
std::function<void(Foo&)> func1 = &Foo::print;
std::function<void(Foo&, int)> func2 = &Foo::add;
```

在这里我们想要包装 `Foo` 类的成员函数，所以第一项参数就为 `Foo&`。

随后在调用时，第一项参数也必须传入对应的类型对象实例，表示调用这个对象的相应成员函数。

```c++
func1(foo);
func2(foo, 1);
```

## std::bind

`std::bind` 可以把可调用对象和某些参数进行绑定，来产生一个新的可调用对象，绑定后的结果也可以用`std::function` 进行保存。参考下面的例子：

```c++
#include <stdio.h>
#include <functional>

int multiply(int a, int b, int c)
{
    printf("a=%d b=%d c=%d\n", a, b, c);
    return a * b * c;
}

int main()
{
    auto f1 = std::bind(multiply, 10, 20, 30);
    printf("%d\n", f1());
    // f1() == multiply(10, 20, 30)

    using namespace std::placeholders; // for _1, _2, _3...

    auto f2 = std::bind(multiply, 5, _1, _2);
    printf("%d\n", f2(2, 3));
    // f2(x, y) == multiply(5, x, y)

    auto f3 = std::bind(multiply, _1, _2, 5);
    printf("%d\n", f3(2, 3));
    // f3(x, y) == multiply(x, y, 5)

    auto f4 = std::bind(multiply, _1, _1, _2);
    printf("%d\n", f4(2, 3));
    // f4(x, y) == multiply(x, x, y)

    auto f5 = std::bind(multiply, _3, _2, _1);
    printf("%d\n", f5(1, 2, 3));
    // f5(x, y, z) == multiply(z, y, x)

    auto f6 = std::bind(f2, _1, 3);
    printf("%d\n", f6(2));
    // f6(x) == f2(x, 3) == multiply(5, x, 3)

    return 0;
}
```

`std::bind` 的第一个参数是要进行绑定的可调用对象，随后的参数为绑定到这个可调用对象的参数。

比如，`std::bind(multiply, 10, 20, 30)` 表示将 10, 20, 30 分别绑定到 `multiply` 的三个输入参数上，返回的类型为一个 `std::function<int()>` 类型。因此，调用 `f1()` 就等价于调用 `multiply(10, 20, 30)`。这就是所谓的可调用对象的绑定。

在实际使用中，一般使用 `auto` 来声明 `std::bind` 的返回值，以简化代码的编写。

除了像这个 `f1` 一样绑定所有的参数外，还可以使用占位符来绑定一部分的参数。这些占位符定义在 `std::placeholders` 命名空间中，比如 `std::placeholders::_1`，它表示这个位置将在函数调用时，被传入的第一个参数所替代。

以 `auto f2 = std::bind(multiply, 5, _1, _2);` 为例，这样绑定后，`f2(x, y)` 就等价于 `multiply(5, x, y)`。

甚至我们还可以像 `f4` 那样通过重复使用相同的占位符来表示相同的参数输入，或者像`f5`那样通过占位符来调换参数的顺序。

除此之外，已经被绑定而产生的可调用对象也可以再次被绑定，比如例子中的 `f6`，就是把前面绑定产生的 `f2` 再一次进行了绑定。

使用 `std::bind` 绑定成员函数也是可以的，例如：

```c++
#include <stdio.h>
#include <functional>

struct Foo
{
    int _n;

    Foo(int num) : _n(num) {}
    void add(int i) { _n += i; }
};

int main()
{
    using namespace std::placeholders;

    Foo foo(0);

    auto f1 = std::bind(&Foo::add, &foo, _1);
    f1(1); // f1(x) == foo.add(x)
    printf("%d\n", foo._n);

    auto f2 = std::bind(&Foo::add, &foo, 10);
    f2(); // f2() == foo.add(10)
    printf("%d\n", foo._n);

    return 0;
}
```

## std::mem_fn

C++11 中还提供了一个专门用于包装成员函数的包装器 `std::mem_fn`。它的使用参考下面的例子：

```c++
#include <stdio.h>
#include <functional>

struct Foo
{
    int _n;

    Foo(int num) : _n(num) {}
    void add(int i) { _n += i; }
    void print() { printf("%d\n", _n); }
};

int main()
{
    Foo foo(0);

    auto f1 = std::mem_fn(&Foo::add);
    f1(foo, 1); // == foo.add(1)

    auto f2 = std::mem_fn(&Foo::print);
    f2(foo);    // == foo.print()

    return 0;
}
```

既然 `std::function` 完全可以包装成员函数，那为什么还引入了一个 `std::mem_fn` 呢？这是因为在一些情况下使用 `std::function` 不方便，声明时必须指定可调用对象的函数签名，并且无法使用 `auto` 来声明，比如：

```c++
Foo foo(0);

std::function<void(Foo&, int)> ff1 = &Foo::add;
ff1(foo, 1);

auto ff2 = &Foo::print;
ff2(foo, 1); // error
(foo.*fn)();
```

声明 `ff1` 时，必须完整的写出可调用对象的类型签名，当成员函数参数类型较为复杂时，写起来就会非常繁琐。而如果用 `auto` 来声明，如 `ff2`，产生的又会是一个普通的成员函数指针类型。这种情况下，就可以使用 `std::mem_fn` 来解决。

参考：

- 《深入C++11：代码优化与工程应用》 机械工业出版社
- https://en.cppreference.com/w/cpp/utility/functional/function
- https://en.cppreference.com/w/cpp/utility/functional/bind
- https://en.cppreference.com/w/cpp/utility/functional/bind