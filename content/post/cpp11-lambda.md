---
title: 'C++11：lambda 表达式'
slug: cpp11-lambda
date: 2016-09-20
tags: [C++]
---

## 前言

lambda 这个名字初看起来不明其意，它其实是希腊字母 `λ` 的英文名称，在计算机科学领域，lambda 通常表示匿名函数，它来源于函数式编程的概念。

从 C++11 开始，引入了 lambda 表达式的概念，用来定义匿名函数，lambda 表达式也成为了 C++11 最常用的一个特性之一。

先来看一个简单例子：

```c++
#include <vector>
#include <iostream>
#include <algorithm>

int main()
{
    int count = 0;
    auto count_odd = [&count](int n){
        if (n % 2)
        {
            count++;
            std::cout << n << std::endl;
        }
    };

    std::vector<int> nums{ 1, 2, 3, 4, 5, 6, 7, 8 };
    std::for_each(nums.begin(), nums.end(), count_odd);
    std::cout << "odd number: " << count << std::endl;
    return 0;
}
```

在 C++11 之前，当我们使用 `std::for_each` 遍历一个范围时，第三个参数通常需要传递一个函数指针或者仿函数对象。但在这个例子中，出现了一种全新的写法，我们声明并定义了一个名为 `count_odd` 的对象，并将它作为第三个参数传递到 `for_each` 中。这个 `count_odd` 对象，就是一个所谓的 lambda 函数，在这个例子中它的作用是统计奇数的个数，并把它们打印出来。

## lambda基本语法

lambda 表达式的基本语法如下：

```
[ captures ] ( params ) -> return-type { body }
```

其中：

captures 表示捕获列表，lambda 函数默认情况下不能访问其外部的变量，如果需要访问函数外部的变量，就需要通过这个捕获列表指定，但就算捕获列表为空，这一对中括号 `[]` 也需要写出来。捕获列表的具体写法我们在后面会详细描述。

param 为函数的参数列表，和普通函数参数的写法相同。如果没有任何参数，可以和这一对小括号一起省略。

return-type 为函数的返回类型，lambda 表达式都是使用后置返回类型的形式。如果没有返回值，可以连同 `->` 符号一同省略。除此之外，如果返回值类型十分明确，比如 `return` 语句只有一个，也可以省略这部分，让编译器自动推导返回类型。

body 为这个匿名函数的函数体。

来看一些简单的例子：

```c++
auto f1 = []{ printf("hello"); };
auto f2 = [](int a) { printf("%d", a); };
auto f3 = [](int a, int b) { return a + b; };
auto f4 = [](int a, int b) -> double { return a * b; };
```

## lambda 捕获列表

前面提到，lambda 函数要想访问外部的变量，需要通过捕获列表指定，捕获列表有下面几种形式：

- `[]` 不捕获任何变量
- `[a]` 以值传递的形式捕获变量 a
- `[&a]` 以引用传递的形式捕获变量 a
- `[this]` 以值传递的形式捕获当前 this 指针
- `[=]` 以值传递的形式捕获所有外部变量，包括 this
- `[&]` 以引用传递的形式捕获所有外部变量，包括 this

以上这些形式也可以再进行一些组合，表示更复杂一些的含义，比如：

- `[a, b]`以值传递的形式捕获变量 a 和 b
- `[a, &b]`按值捕获 a，按引用捕获 b
- `[=, &a, &b]`按引用捕获 a 和 b，按值捕获其他变量
- `[&, a, b]`按值捕获 a 和 b，按引用捕获其他变量

回头再看看最开始的例子中 `count_odd` 的定义，

```c++
auto count_odd = [&count](int n){
    if (n % 2)
    {
        count++;
        std::cout << n << std::endl;
    }
};
```

它通过引用的方式捕获了外部的 `count` 变量，因此就可以在lambda内部修改这个 `count` 变量。它接收一个 int 类型的参数，并在内部判断其是否为奇数。因为这个 lambda 函数没有返回值，所以在定义时省略了返回值类型的声明。

这里需要说明一下 lambda 表达式的类型。在 C++11 中，lambda 表达式并非传统的函数指针或者仿函数类型，而是一种特殊的，称之为闭包（Closure）的类型。因此，在定义一个 lambda 对象时，通常情况下需要使用 `auto` 自动推导来声明其类型。不过，很多时候 lambda 表达式确实可以被转换为函数指针或者 `std::function` 对象。

## 使用 lambda 简化代码逻辑

在很多情况下，使用 lambda 可以简化代码逻辑，比如和 STL 的一些算法函数结合的时候。

```c++
#include <vector>
#include <algorithm>
#include <iostream>

int main()
{
    auto print = [](int a){ std::cout << a << " "; };

    std::vector<int> nums = { -4, -3, -2, -1, 0, 1, 2, 3, 4 };
    std::for_each(nums.begin(), nums.end(), print);
    std::cout << std::endl;
    // -4 -3 -2 -1 0 1 2 3 4

    std::sort(nums.begin(), nums.end(), [](int a, int b){ return std::abs(a) < std::abs(b); });
    std::for_each(nums.begin(), nums.end(), print);
    std::cout << std::endl;
    // 0 -1 1 -2 2 -3 3 -4 4

    std::sort(nums.begin(), nums.end(), [](int a, int b){ return std::abs(a) > std::abs(b); });
    std::for_each(nums.begin(), nums.end(), print);
    std::cout << std::endl;
    // -4 4 -3 3 -2 2 -1 1 0

    int max = 2;
    int count = std::count_if(nums.begin(), nums.end(), [max](int n){ return n < max; });
    std::cout << count << " numbers are less than " << max << std::endl;
    // 6 numbers are less than 2

    return 0;
}
```

在这个例子中，我们先后调用了 STL 中的 `std::count_if`, `std::sort`, `std::for_each`，这些函数都有一个共同点，它们都需要一个“函数对象”类型的参数。在以前，一般需要通过函数指针或者仿函数的形式来实现，在这里我们使用 lambda 函数的方式来实现。

在调用 `std::for_each` 时，我们传入了一个提前定义的名为 `print` 的lambda函数对象，用来打印输出 vector 中的所有元素。在调用 `std::sort` 时，稍微有一些不同，这里并没有提前声明定义一个 lambda 对象，而是直接就地定义并传递了一个临时的 lambda 对象，它接受两个 int 类型参数，并比较它们的绝对值。通过这样一个 lambda 函数，我们实现了对 nums 中的元素进行绝对值大小排序。在 `std::count_if` 的调用中，同样也是传递了一个临时的 lambda 对象，并且捕获了一个 `max` 变量用于比较大小。

通过这些例子可以看到，使用 lambda 能够简化代码的编写，使逻辑也更加直观简洁清晰。

## 捕获列表注意事项

有关 lambda 的捕获列表，还有一些事情需要注意。

捕获列表中不允许有重复或者冲突的内容。比如以下这几个写法都是不允许的：

```c++
[a, a]   // 重复
[a, &a]  // 冲突
[=, a]   // 重复，= 已经表示按值捕获所有变量
[&, &a]  // 重复，& 已经表示按引用捕获所有变量
```

在捕获列表中，如果出现 `[=]`，并不是真的复制了所有的外部变量到 lambda 函数内部中，实际上只复制了在 lambda内部使用到的那些变量，所以不必担心存在额外的变量复制会影响运行效率。`[&]` 也是同理，它只引用使用到的变量。

lambda 函数的变量捕获，指的是局部变量的捕获，对于非局部变量，如果是全局变量，静态变量或者是 namespace 中定义的变量，则不需要捕获就可以直接使用，如果是类成员变量，则不能直接捕获，只能通过捕获 `this` 的方式访问。例如，

```c++
int a = 0;
namespace FOO { int b = 0; }
const char s[] = "hello";

int main()
{
    auto f1 = [](){ a++; }; // 直接修改a
    auto f2 = [](){ FOO::b++; }; // 直接修改FOO::b
    auto f3 = [](){ std::cout << s << std::endl; }; // 直接访问s

    f1();
    f2();
    f3();

    std::cout << a << " "<< FOO::b << std::endl; // 输出 1 1

    return 0;
}
```

```c++
class Foo
{
    int a = 0;

    void do_something()
    {
        auto f = [this](){ printf("%d", a); }; // OK
        //auto f = [](){ printf("%d", a); }; // Error
        //auto f = [&a](){ printf("%d", a); }; // Error
        f();
    }
};
```

另外，在块代码外部定义的 lambda 函数不能有捕获列表，例如，

```c++
auto gf = [](){ printf("1"); }; // 代码块外部定义，捕获列表必须为空
int main()
{
    gf();
    return 0;
}
```

这一点很好理解，因为 `gf` 所在的地方没有任何局部变量可以捕获。一般情况也完全没必要这么定义 lambda 函数，直接写普通函数就好了。

有关捕获的时间。变量的捕获在 lambda 函数被定义的时候就确定了。因此，对于按值捕获的变量，捕获的值就是 lambda 函数被定义时的值，而不是 lambda 被调用时的值。比如，

```c++
int main()
{
    int a = 0;

    auto f1 = [a](){ printf("a = %d\n", a); };
    auto f2 = [&a](){ printf("a = %d\n", a); };

    a++;

    f1(); // 输出 a = 0;
    f2(); // 输出 a = 1;

    return 0;
}
```

在这个例子中，当定义 `f1` 时，它函数内部那个 `a` 的值就已经确定了为0，即使后面在 `f1` 被调用之前又修改了 `a`，也不会改变 `f1` 内的 `a` 的值。但 `f2` 是按引用捕获的 `a`，所以在修改 `main` 中的 `a` 后，也会影响 `f2` 中 `a` 的值。因此，在定义 lambda 函数时，需要根据捕获变量的使用需求，正确选择按值捕获还是按引用捕获。

## lambda 与仿函数

再来看一下本文最开始的那个例子，如果不使用 lambda，要实现类似的功能，也可以使用仿函数来写：

```c++
class CountOdd
{
private:
    int& count;

public:
    CountOdd(int& c) : count(c){ }

    void operator()(int n)
    {
        if (n % 2)
        {
            count++;
            std::cout << n << std::endl;
        }
    }
};

int main()
{
    int count = 0;
    CountOdd count_odd(count);
    std::vector<int> nums{ 1, 2, 3, 4, 5, 6, 7, 8 };
    std::for_each(nums.begin(), nums.end(), count_odd);
    std::cout << "odd number: " << count << std::endl;
    return 0;
}
```

在这段代码中，`class CountOdd` 就是我们通常所说的仿函数，也就是重新定义了 `operator()` 的一种自定义类型。可以看到，使用 `main` 中的 `count` 来初始化 `CountOdd` 中的 `count`，就类似 lambda 中的捕获，而 `CountOdd` 的 `operator()` 函数，与之前 lambda 的函数体也是类似的。所以，这种使用仿函数的写法，和之前 lambda 的写法其实是逻辑等价的，仿函数 `CountOdd` 对象和之前的 lambda 函数对象 `count_odd` 也是逻辑等价的。

因此，在 C++11 中，完全可以将 lambda 函数视为仿函数的一种等价形式，或者可以说，lambda 就是传统仿函数的一种“语法糖”。并且实际上仿函数也是编译器实现 lambda 函数的一种方式。

理解和想象出一个 lambda 的等价仿函数形式，可以更容易的理解 lambda 的一些特性。比如前面提到的：对于按值捕获的变量，捕获的值就是 lambda 函数被定义时的值，而不是 lambda 被调用时的值。如果我们将那个例子中的 lambda 换成仿函数的形式，就会发现这是其实是自然而然的：

```c++
class F1
{
    int a;
public:
    F1(int n) : a(n) { }
    void operator()() { printf("a = %d\n", a); }
};

class F2
{
    int &a;
public:
    F2(int &n) : a(n) { }
    void operator()() { printf("a = %d\n", a); }
};

int main()
{
    int a = 0;
    F1 f1(a); // 等价于 auto f1 = [a](){ printf("a = %d\n", a); };
    F2 f2(a); // 等价于 auto f2 = [&a](){ printf("a = %d\n", a); };
    a++;
    f1(); // 输出 a = 0;
    f2(); // 输出 a = 1;
    return 0;
}
```

## lambda 函数 mutable 属性

默认情况下，lambda 函数都是 `const` 的，或者可以理解为，与之等价的仿函数类的 `operator()` 成员函数是被 `const` 修饰的。比如这个 lambda 函数：

```c++
int a = 0;
auto f1 = [a](){ printf("a = %d\n", a); };
```

等价于：

```c++
class F1
{
    int a;
public:
    F1(int n) : a(n) { }
    void operator()() const { printf("a = %d\n", a); }
};
```

注意这个例子相比之前，仅仅是将 `operator()` 成员增加了 `const` 的限定（其实本文之前所有的等价仿函数都应当添加 `const` 限定）。这就意味着，对于按值捕获的变量，在 lambda 函数体内不允许对其进行修改。

```c++
int a = 0;
auto f1 = [a](){ a++; }; // 错误
```

因为

```c++
void operator()() const { a++; } // 错误
```

如果在定义 lambda 时添加 `mutable` 关键字，则可以取消这种 `const` 属性。比如，

```c++
int main()
{
    int a = 0;
    auto f = [a]() mutable {
        printf("a = %d\n", a);
        a++;
    };

    f(); // 输出 a = 0
    f(); // 输出 a = 1
    f(); // 输出 a = 2

    printf("the 'a' in main is %d\n", a); // the 'a' in main is 0

    return 0;
}
```

这里的 lambda 函数 `f` 被声明为 `mutable` 的，所以它可以修改其中的 `a`。但一定注意，这里的修改的是 lambda 函数内部的那个 `a`，而不是 `main` 中的那个 `a`。如果你有点晕，再看一下上一节的内容，然后想象一下与之等价的仿函数对象，就能分清这两个 `a` 了。

使用 `mutable` 确实可以取消 lambda 函数的 const 属性，但这只是提供了语法上的可行性，修改按值捕获的变量并没有什么实际意义，现实中很少会使用到这种方式。

对于按引用捕获的变量则不受 lambda 函数 const 属性的影响，使用同样的办法，想象一个等价的仿函数就能明白其中的原因：

```c++
class F2
{
    int &a;
public:
    F2(int &n) : a(n) { }
    void operator()() const { a++; } // 完全允许
};
```