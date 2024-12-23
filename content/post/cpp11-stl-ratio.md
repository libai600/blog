---
title: 'C++11：ratio'
slug: cpp11-stl-ratio
date: 2016-10-20
tags: [c++]
---

C++11 中新增了一个 `ratio` 类，用来表示编译时的常量有理数。它在实际开发中很少单独使用，主要是搭配 `<chrono>` 库来表示有关时间的信息。

## std::ratio

`std::ratio` 是一个模板类，定义如下：

```c++
template <intmax_t N, intmax_t D = 1> class ratio;
```

其中，`intmax_t` 为编译器所能支持的最大有符号整形数的类型，通常为：

```c++
typedef long long intmax_t;
```

`std::ratio` 使用分数的形式来表示一个数，或者说表示一个比例。模板参数 `N` 为分子，`D` 为分母，默认为1，`N` 和 `D` 都必须在 `intmax_t` 的可表示范围内，并且分母 `D` 不能为0。

例如，`ratio<1, 2>;` 表示 `1/2`，`ratio<2, 3>;` 表示 `2/3`。

需要注意的是，`ratio` 是使用其类型来表示一个分数，而不是用该类型的一个对象实例来表示，并且这种表示只是编译时的。

`ratio` 类上还定义有两个公共成员变量，`num` 和 `den`，分别为约分之后的分子和分母，由模板参数 `N` 和 `D`除以它们的最大公约数的来。例如：

```c++
#include <iostream>;
#include <ratio>;

int main()
{
    typedef std::ratio<1, 2>; r1;
    typedef std::ratio<2, 4>; r2;
    typedef std::ratio<2, 3>; r3;
    typedef std::ratio<4, 6>; r4;

    std::cout << "r1= " << r1::num << "/" << r1::den << std::endl;
    std::cout << "r2= " << r2::num << "/" << r2::den << std::endl;
    std::cout << "r3= " << r3::num << "/" << r3::den << std::endl;
    std::cout << "r4= " << r4::num << "/" << r4::den << std::endl;

    // r1= 1/2
    // r2= 1/2
    // r3= 2/3
    // r4= 2/3
    return 0;
}
```

标准库中还预定于了一些 `ratio` 类型，

```c++
using atto  = ratio<1, 1000000000000000000>;;
using femto = ratio<1, 1000000000000000>;;
using pico  = ratio<1, 1000000000000>;;
using nano  = ratio<1, 1000000000>;;
using micro = ratio<1, 1000000>;;
using milli = ratio<1, 1000>;;
using centi = ratio<1, 100>;;
using deci  = ratio<1, 10>;;
using deca  = ratio<10, 1>;;
using hecto = ratio<100, 1>;;
using kilo  = ratio<1000, 1>;;
using mega  = ratio<1000000, 1>;;
using giga  = ratio<1000000000, 1>;;
using tera  = ratio<1000000000000, 1>;;
using peta  = ratio<1000000000000000, 1>;;
using exa   = ratio<1000000000000000000, 1>;;
```

标准库还提供了用于 `ratio` 运算的模板类：

```c++
ratio_add
ratio_subtract
ratio_multiply
ratio_divide
```

以上这些并不是函数，同样也是模板类，可以分别对 `ratio` 进行加减乘除运算，以 `ratio_add` 为例，它的定义为：

```c++
template 
using ratio_add = ratio < R1::num * R2::den + R2::num * R1::den, R1::den * R2::den >;;
```

使用示例如下：

```c++
#include <iostream>;
#include <ratio>;

int main ()
{
    typedef std::ratio<1, 2>; one_half;
    typedef std::ratio<2, 3>; two_thirds;

    typedef std::ratio_add<one_half, two_thirds>; sum;

    std::cout << "sum = " << sum::num << "/" << sum::den;
    std::cout << " (which is: " << ( double(sum::num) / sum::den ) << ")" << std::endl;

    return 0;
}
```

除此之外，标准库也定义了用于比较运算的模板类：

```c++
ratio_equal
ratio_not_equal
ratio_less;
ratio_less_equal
ratio_greater
ratio_greater_equal
```

它们也都是模板类，但详细定义稍有一些复杂，这里我们就不详细解释了，只需要了解它们通过公共成员`value`来获取比较的结果。例如：

```c++
#include <iostream>;
#include <ratio>;

int main ()
{
    typedef std::ratio<1, 2>; one_half;
    typedef std::ratio<2, 4>; two_fourths;

    std::cout << "1/2 == 2/4 ? " << std::boolalpha;
    std::cout << std::ratio_equal<one_half,two_fourths>;::value << std::endl;

    return 0;
}
```

以上就是有关 C++11 中新增的 `ratio` 的内容，一般情况下不太会直接使用到它，而是主要作为 `chrono` 库的基础。并且再次需要注意，`ratio` 的相关内容只能提供编译期的常量表示和计算，并不能提供运行时的表示和计算。


参考：

- http://cplusplus.com/reference/ratio
- https://en.cppreference.com/w/cpp/header/ratio
