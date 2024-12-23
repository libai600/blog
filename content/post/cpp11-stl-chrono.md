---
title: 'C++11：chrono'
slug: cpp11-stl-chrono
date: 2016-11-01
tags: [C++]
---

C++11 新增了一个和时间日期相关的模块`chrono`，位于头文件`<chrono>`中，用来处理有关时间日期的表示和计算，功能十分强大。它主要包括以下三个概念：

- **时间段**（Durations）：顾名思义就是一段时间，比如十毫秒，两分钟，三个小时等等。
- **时间点**（Time points）：表示具体一个时间，比如2016年9月24号上午11点整。
- **时钟**（Clocks）：和实际时间相关的时钟，比如当前系统的时钟。


需要注意，`<chrono>` 的相关类和函数并不是定义在 `namespace std` 中，而是在 `namespace std::chrono` 中。

## 时间段 Durations

我们所说的“一段时间”的概念，包括了两个部分：数量和周期单位，比如 100 毫秒这一段时间，单位为毫秒，数量为 100，即 100 个周期单位，一个周期为 1 毫秒。在很多外文资料中，这个数量通常又被称为 Tick，周期单位称为 Period，100 毫秒即为 100 个 tick，每个 tick 的 Peroid 为 1 毫秒。

在 C++11 中，时间段使用 `std::chrono::duration` 类来表示。它的定义简化后大概如下：

```c++
template < class Rep, class Period = ratio<1> >
class duration
{
private:
    Rep _count;
public:
    explicit duration(const Rep&; val) : _count(val) { }
    Rep count() const { return _count; }
};
```



其中，模板参数 `Period` 即为周期单位，它使用 [`std::ratio`]({{< ref "/post/cpp11-stl-ratio.md" >}}) 类型来表示。而这里的 `ratio`，又以 1 秒为基准，比如 `ratio<1>` 表示周期为 1 秒，`ratio<60>` 表示周期为 60 秒即 1 分钟，`ratio<1,1000>` 表示周期为千分之一秒，即 1 毫秒，以此类推。

时间段的数量由成员变量 `_count` 表示，也就该时间段包含多少个周期单位，它需要在构造时指定，并可以通过成员函数 `count()` 获取。而模板参数 `Rep` 是用来表示这个数量的类型，通常设为整形，比如 `int` 或者 `long long`。

来看下面这个例子：

```c++
#include <chrono>
#include <iostream>

int main()
{
    using namespace std;
    using namespace std::chrono;

    typedef duration<int, ratio<1>> sec_t;  // 表示秒的类型
    typedef duration<int, ratio<60>> min_t; // 表示分钟的类型

    // or
    // using sec_t = duration<int, ratio<1>>;
    // using min_t = duration<int, ratio<60>>;

    sec_t ten_seconds(10); // 10秒
    min_t two_minutes(2);  // 2分钟

    std::cout << ten_seconds.count() << std::endl; // 10
    std::cout << two_minutes.count() << std::endl; // 2

    return 0;
}
```

因为 `duration` 是一个类模板，为了方便使用，通常会先定义好一些类型别名。在这里我们定义了两个表示时间段的类型，`sec_t` 和 `min_t`，分别表示秒和分钟。然后实例化了两个对象 `ten_seconds` 和 `two_minutes` 来分别表示十秒和两分钟。这里的 `sec_t` 和 `min_t` 都是使用 `int` 类型来存储数据，当然，如果你需要表示一个非常大的时间段，也可以定义为使用 `long long` 来表示。

可以看出，当模板参数 `Rep` 为一个整型类型时，`Period` 实际上就是该 `duration` 代表时间段类型的精度，`sec_t` 的表示精度就是1秒，`min_t` 的表示精度就是1分钟，你不可能使用 `sec_t` 来表示出毫秒的概念，也不可能使用 `min_t` 表示出秒的概念。这种精度的规则仅限于 `Rep` 为一个整型类型时，理论上你确实可以把 `Rep` 设置为 `double` 类型，从而可以用 `min_t` 表示出 0.5 分钟甚至零点几秒的时间段，但一般情况下不太推荐这么做，应当通过调整 `ratio` 的参数来调整时间段的表示精度。

为了方便使用，在 `<chrono>` 中其实已经预定义好了几种常见的时间单位类型：

```c++
using nanoseconds  = duration<long long, ratio<1, 1000000000>>; // 纳秒
using microseconds = duration<long long, ratio<1, 1000000>>;    // 微秒
using milliseconds = duration<long long, ratio<1, 1000>>;       // 毫秒
using seconds      = duration<long long, ratio<1>>; // 秒
using minutes      = duration<int, ratio<60>>;      // 分钟
using hours        = duration<int, ratio<3600>>;    // 小时
```

不同单位的时间段类型可以通过 `std::chrono::duration_cast` 来互相转换，但如果转换后的时间精度不足以表示，精度就会丢失。例如：

```c++
#include <chrono>
#include <iostream>

int main()
{
    using namespace std;
    using namespace std::chrono;

    auto p1 = chrono::seconds(5); // 5秒
    auto p2 = chrono::milliseconds(2500); // 2500毫秒

    auto p3 = duration_cast<chrono::milliseconds>(p1); // 5000毫秒
    auto p4 = duration_cast<chrono::seconds>(p2);      // 2秒，精度丢失

    std::cout << p3.count() << std::endl; // 5000
    std::cout << p4.count() << std::endl; // 2

    return 0;
}
```

实际上，很多情况下也可以不使用 `duration_cast`，`duration` 不同类型之间是可以进行隐式转换的，例如：

```c++
auto p1 = chrono::hours(5);
chrono::seconds p2 = p1;
chrono::milliseconds p3 = p1;
p3 = chrono::seconds(100);
```

`duration` 也支持基本的运算操作，包括加减乘除，自增自减，和比较运算。看下面的例子：

```c++
#include <chrono>
#include <iostream>

int main()
{
    using namespace std;
    using namespace std::chrono;

    auto p1 = chrono::seconds(2);        // 2s
    auto p2 = chrono::milliseconds(500); // 500ms

    chrono::milliseconds p3 = p1 + p2;   // 2500ms
    auto p4 = p1 - p2;                   // 1500ms

    std::cout << p3.count() << std::endl;
    std::cout << p4.count() << std::endl;

    p1 = p1 * 4; // 8s
    std::cout << p1.count() << std::endl;
    p1 = p1 / 2; // 4s
    std::cout << p1.count() << std::endl;
    p1--;        // 3s
    std::cout << p1.count() << std::endl;

    std::cout << std::boolalpha;
    std::cout << (p3 > p4) << std::endl; // true
    std::cout << (p2 > p1) << std::endl; // false

    return 0;
}
```

`duration` 可以和另一个 `duration` 对象相加减，不同精度单位的对象也可以，就比如上例中的 `p1` 和 `p2`，但其实会先自动统一成两者中精度高的那一种，因此运算结果的类型会是两者中精度高的。`duration` 可以乘以或者除以一个整形，表示该时间段的倍数。`duration` 对象之间也可以进行比较，包括不同精度对象之间的比较也可以。

## 时钟 Clocks

时钟和时间点的概念联系比较紧密，我们先来介绍时钟的内容。

首先需要明确时钟也具有一个精度的属性。想象一下常见的挂钟和机械手表，基本表盘上都会有个秒针，它们所能体现的最大精度就是秒，但也可能一些挂钟没有秒针，那它的精度就是分，又比如很多运动手表都能显示出毫秒的单位，那它的精度就是毫秒。

`chrono` 中定义了三种类型的时钟：`system_clock`，`steady_clock`，`high_resolution_clock`。我们先来介绍 `system_clock`。

### system_clock

`system_clock` 用来表示当前的系统时钟，也就是系统上的真实时间。它的定义大概如下：

```c++
struct system_clock
{
    typedef long long                         rep;
    typedef ratio<1, 1000000000>              period; // 1纳秒
    typedef chrono::duration<rep, period>     duration;
    typedef chrono::time_point<system_clock>  time_point;

    static time_point now();
    static std::time_t to_time_t(const time_point&; time);
    static time_point from_time_t(std::time_t tm);

    static constexpr bool is_steady = false;
};
```

可以看到在它的结构内定义有四个类型属性，其中 `system_clock::period` 就是这个 `system_clock` 的精度，同样使用 `std::ratio` 类型来表示，这个精度在不同的环境中可能存在差异，在笔者所用的 GCC 环境中，该精度为 1 纳秒，但在 MSVC 环境中，该精度为 100 纳秒。

剩下的三个类型属性中：

`system_clock::rep` 为该时钟内部数据的存储类型，与前面提到的 `duration` 的模板参数 `Rep` 类似。
`system_clock::duration` 为这个时钟上所能表示的时间段类型。
`system_clock::time_point` 为这个时钟上所能表示的时间点的类型，它的类型为 `chrono::time_point`，这个类型在后面会详细解释，这里暂时只需要知道用它来表示该时钟上的时间点。

`system_clock` 提供了三个静态函数：

`now()`，它返回一个 `system_clock::time_point` 类型的对象，用来获取当前的时间点。

`to_time_t()`，它可以将一个 `system_clock::time_point` 对象转换为 `time_t` 类型，如果我们需要打印输出一个 `system_clock::time_point` 时间点信息，就需要通过这个函数先将其转换为 `time_t` 类型再进行打印。

`from_time_t()` 则与 `to_time_t` 相反，它可以把一个 `time_t` 类型转换为 `time_point` 类型。

来看下面这个简单例子：

```c++
#include <chrono>
#include <iostream>
#include <ctime>

int main ()
{
    using namespace std::chrono;
    auto now = system_clock::now();
    time_t tt = system_clock::to_time_t(now);
    std::cout << ctime(&;tt) << std::endl;
    return 0;
}
```

在 `system_clock` 上还定义了一个 `is_steady` 的静态类型，对于 `system_clock` 这个类型的时钟来说，它为 `false`，我们稍后再解释它的含义。

### steady_clock

先再次看一下 `system_clock` 的定义，在最后面它包含一个 `is_steady` 的静态常量，并且值为 `false`，这个常量表示 `system_clock` 不是一个“稳定的”时钟。这里的“稳定”指的是时钟只能顺时针往前走，不能逆时针往后倒，也就是说，在一个程序内，先由 `system_clock::now()` 获取的时间点必须始终早于之后再次由 `system_clock::now()` 获取的时间点。但实际上，`system_clock` 所代表的是系统时间，如果在程序运行过程中人为的修改系统时间，就不能满足这个“稳定”的要求，因此`system_clock` 不是稳定的。也正是由于这一点，如果想计算某段代码执行所需要的时间，就不能使用 `system_clock`。

所以，`chrono` 提供了另外一个时钟类型：`steady_clock`，而它就是一种“稳定的”时钟。先来看一下它的定义：

```c++
struct steady_clock
{
    typedef long long                         rep;
    typedef ratio<1, 1000000000>              period;
    typedef chrono::duration<rep, period>     duration;
    typedef chrono::time_point<system_clock>  time_point;

    static time_point now();

    static constexpr bool is_steady = false;
};
```

可以看到，在它的结构能同样定义了 `rep`, `period`, `duration`, `time_point` 四种类型，它们的含义和 `system_clock` 内部的类型相同，这里不再赘述。其中 `period` 精度的定义在不同环境中有可能不同，在同一环境中与 `system_clock` 精度定义也可能不同。

在 `steady_clock` 内同样定义了一个 `is_steady` 静态常量，值为 `true`，表示它是一个稳定的时钟。

静态函数 `steady_clock::now()` 用来获取该时钟的当前时间点。但与 `system_clock` 相比，`steady_clock` 少了 `to_time_t` 和 `from_time_t` 两个函数，这是因为，虽然 `steady_clock` 是一个稳定的时钟类型，但它是一个虚拟的时钟，它不对应系统的时钟，它上面的时间点和实际时间也没有对应关系。

其实 `steady_clock` 就是为了计算程序执行时间间隔而设计的，如果想获取实际的系统时间，应当使用 `system_clock`，而如果想计算程序执行时间，应当使用 `steady_clock`，这一点其实类似于传统的 `time.h` 中 `time()` 和 `clock()` 的区别。

来看 `steady_clock` 的应用例子：

```c++
#include <chrono>
#include <iostream>

int main ()
{
    using namespace std;
    using namespace std::chrono;

    steady_clock::time_point t1 = steady_clock::now();

    for (int i = 0; i < 1000; ++i)
        std::cout << "Hi, there." << std::endl;

    steady_clock::time_point t2 = steady_clock::now();

    chrono::nanoseconds time_span = duration_cast<chrono::nanoseconds>(t2 - t1);
    std::cout << time_span.count() << " nanoseconds." << std::endl;
    return 0;
}
```

从这个例子中可以看出，要计算时间间隔，只需要将两个时间点相减即可。

### high_resolution_clock

除了 `system_clock` 和 `steady_clock` 以外，`chrono` 中还提供了一种 `high_resolution_clock`，它表示当前环境中所能提供的最高精度的时钟，但标准并未规定它应该达到多大的精度，或者它是否要稳定。基本上它就是 `system_clock` 或者 `steady_clock` 的别名。

```c++
using high_resolution_clock = system_clock;
// Or using high_resolution_clock = steady_clock;
```

通常情况下，直接使用 `steady_clock` 和 `system_clock` 就可以了，计算时间过了多久用 `steady_clock`，获取实际时间用`system_clock`，一般用不着 `high_resolution_clock`。

## 时间点 Time points

时间点通过 `chrono::time_point` 类来表示，时间点的概念是和时钟相关的，表示的是某个时钟上的某个时间点，这一点也体现在 `chrono::time_point` 的结构定义上，它的简化定义如下：

```c++
template <class Clock, class Duration = typename Clock::duration>
class time_point
{
public:
    using clock    = Clock;
    using duration = Duration;
    using rep      = typename Duration::rep;
    using period   = typename Duration::period;

    Duration time_since_epoch() const { return _d; }

private:
    Duration _d { Duration::zero() }; // duration since the epoch
};
```

其中，模板参数 `Clock` 就是其所关联的时钟类型。回头看一下前面 `system_clock` 结构中 `time_point` 类型的定义，就容易理解了。

```c++
struct system_clock
{
    ...
    typedef chrono::time_point<system_clock> time_point;
    ...
};
```

`time_point` 其实是通过内部的一个时间段变量 `_d` 来表示一个具体的时间点，这个时间段成员 `_d` 表示的是从时钟的原点（epoch）开始，到现在所经过的时间长度。对于 `system_clock` 来说，这个时钟原点通常就是1970年1月1日，对于 `steady_clock` 来说，通常是系统上次启动的时间，或者程序开始运行的时间。成员函数 `time_since_epoch()` 用来获取自epoch开始到这个时间点的长度。通过默认函数构造的 `time_point` 对象就表示的是epoch时间。来看下面这个例子：

```c++
#include <chrono>
#include <iostream>
#include <ctime>

int main()
{
    using namespace std;
    using namespace std::chrono;

    system_clock::time_point epoch;
    time_t t_epoch = system_clock::to_time_t(epoch);
    std::cout << "epoch: " << ctime(&;t_epoch);

    system_clock::time_point now = system_clock::now();
    time_t t_now = system_clock::to_time_t(now);
    std::cout << "now: " << ctime(&;t_now);

    std::cout << duration_cast<chrono::hours>(now.time_since_epoch()).count() << " hours since epoch" << std::endl;

    using day_t = duration<int, ratio<24*60*60>>;
    std::cout << duration_cast<day_t>(now.time_since_epoch()).count() << " days since epoch"<< std::endl;

    return 0;
}
```

时间点 `time_point` 也支持基本的运算，它可以和一个时间段相加减，得到向前或向后移动的时间点，两个时间点可以相减，得到它们的时间差，时间点之间也可以进行比较。参考下面这个例子：

```c++
#include <iostream>
#include <chrono>

int main()
{
    using namespace std::chrono;

    system_clock::time_point tp, tp2;        // epoch value
    system_clock::duration dtn(seconds(1));  // 1 second

                        //  tp     tp2    dtn
                        //  ---    ---    ---
    tp += dtn;          //  e+1s   e      1s
    tp2 -= dtn;         //  e+1s   e-1s   1s
    tp2 = tp + dtn;     //  e+1s   e+2s   1s
    tp = dtn + tp2;     //  e+3s   e+2s   1s
    tp2 = tp2 - dtn;    //  e+3s   e+1s   1s
    dtn = tp - tp2;     //  e+3s   e+1s   2s

    std::cout << std::boolalpha;
    std::cout << "tp == tp2: " << (tp==tp2) << std::endl;
    std::cout << "tp > tp2: " << (tp>tp2) << std::endl;
    std::cout << "dtn: " << dtn.count() << std::endl;

    return 0;
}
```

参考：

- http://cplusplus.com/reference/chrono
- https://en.cppreference.com/w/cpp/header/chrono
