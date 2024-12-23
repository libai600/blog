---
title: 'C++11：右值引用和移动语义'
slug: cpp11-rvalue
date: 2016-06-01
tags: [C++]
---

C++11 引入了右值引用和移动语义，可以避免不必要的复制，提高程序性能。

## 左值和右值

首先我们来了解一下什么是左值和右值。

在编程语言的领域中，我们经常会提起左值和右值的概念。一般情况下，在赋值表达式中，出现在等号左边的称为“左值”，出现在等号右边的称为“右值”。但这个概括很不准确，实际上在 C++ 标准中，左值和右值的严格准确定义比较复杂，难以用一两句话归纳。不过通常我们有一个简便的方法来判断：看是否能对其取地址，可以取地址的，就是左值，反之，不能取地址的，则为右值。

例如，

```c++
int i = 0;
int a = b + c;
```

在这个例子中，`i` 和 `a` 是左值，`0` 和 `b+c` 是右值。因为 `&i` 和 `&a` 这种取地址操作是允许的，但 `&0` 和 `&(b+c)` 这样的操作是不允许的。

在 C++11 中，右值由两个概念构成，一个是纯右值（prvalue, pure rvalue），一个是将亡值（xvalue, expiring value）。其中纯右值就是 C++98 中右值的概念，包括例如非引用返回的临时变量，一些运算表达式比如 1+2 产生的临时变量值，原始字面量如 0, true 等，此外，类型转换函数的返回值，lambda 表达式也都是纯右值。将亡值是 C++11 新增的，与右值引用相关的表达式，比如将要被移动的对象，`T&&` 函数返回值，`std::move` 返回值和转换为 `T&&` 的类型的转换函数的返回值。

在 C++11 中，所有的值都属于左值，纯右值，将亡值三者之一。


## 右值引用

**右值引用**（Rvalue reference）就是对一个右值进行引用的类型，与通常的左值引用不同，右值引用使用两个 `&` 表示，例如 `T&&`。因为右值通常不具有名字，所以只能通过引用的方式找到它的存在。通常情况下，我们只能从右值表达式获得其引用。

来看下面这个例子：

```c++
#include <cstdio>

struct Foo
{
    Foo() { printf("constructor %p\n", this); }
    ~Foo() { printf("destructor %p\n", this); }
};

Foo getFoo()
{
    Foo a;
    return a;
}

int main()
{
    Foo&& f = getFoo();
    printf("f address: %p\n", &f);
    return 0;
}
```

程序的运行输出如下：

```
constructor 0x7ffdd2940ec7
f address: 0x7ffdd2940ec7
destructor 0x7ffdd2940ec7
```


函数 `getFoo()` 返回一个非引用的临时量，也就是一个右值，在 `main` 函数中我们声明了一个名为 `f` 的右值引用，并使用 `getFoo` 对其进行初始化，`f` 实际上就是 `getFoo()` 函数返回的临时变量，这一点也可以通过我们输出的地址值看出来。

为了区别于 C++98 中的引用类型，我们称 C++98 中的引用为**左值引用**（lvalue reference）。右值引用和左值引用都属于引用类型，右值引用和左值引用一样，都必须在声明的同时进行初始化。左值引用是具名变量的别名，而右值引用则是不具名（匿名）变量的别名。

在上面的简单例子中，`getFoo()` 函数返回的右值在表达式语句结束后，其生命也就终结了（通常我们也称其具有表达式生命期），而通过右值引用的声明，该右值可以“重获新生”，其生命周期将与右值引用类型变量 `f` 的生命期一样。只要 `f` 还“活着”，该右值临时量将会一直“存活”下去。

通常情况下，右值引用不能绑定到任何左值。比如下面的代码就无法通过编译，因为 `a` 是一个左值。

```c++
int a;
int && i = a; // Error
```

普通的左值引用也不能绑定到任何右值，但常量左值引用可以绑定到右值。例如：

```c++
Foo& b = getFoo(); // Error
const Foo& a = getFoo(); // OK
```

这是因为常量左值引用是个“万能”的引用类型，它可以绑定到常量左值，非常量左值，以及右值。常量左值引用同样可以像右值引用一样将所绑定右值的生命期延长。

有一点需要注意：一个变量的类型是否为右值引用，和它本身是否为一个右值，是两个不同的概念。

比如在刚才的例子中：

```c++
Foo&& f = getFoo();
printf("f address: %p\n", &f);
```

变量 `f` 的类型是右值引用 `Foo&&`，然而对于 `f` 自身而言，它却是一个左值，因为它具有名字，对其取地址是完全合法的，所以在第二行中我们可以输出 `f` 的地址。又比如 `int&& i = 0;` 中，`i` 的类型是`i&&`，它的类型是一个右值引用，但 `i`本身是一个左值，因为对 `i` 取地址 `&i` 是允许的。理解这一点区别非常重要，在后面很多内容中我们还会再次用到它。

## 移动语义

某些情况下，临时的变量会导致额外的复制开销，比如下面这个例子：

```c++
#include <cstdio>
#include <cstring>

struct Foo
{
    char *data;

    Foo(const char* str) : data(new char[strlen(str) + 1])
    {
        strcpy(data, str);
        printf("constructor %p %p\n", this, data);
    }

    Foo(const Foo& other) : data(new char[strlen(other.data) + 1])
    {
        strcpy(data, other.data);
        printf("copy constructor %p %p\n", this, data);
    }

    ~Foo()
    {
        delete [] data;
        printf("destructor %p\n", this);
    }
};

Foo getFoo()
{
    // 故意如此，使得无法进行返回值优化
    if (true)
    {
        Foo a("blablabla");
        return a;
    }
    else
    {
        Foo b("foo");
        return b;
    }
}

int main()
{
    Foo f = getFoo();
    return 0;
}
```

程序运行输出如下：

```
constructor 0x7fff1c5b0bf0  0xdeb010
copy constructor 0x7fff1c5b0c20  0xdeb030
destructor 0x7fff1c5b0bf0
destructor 0x7fff1c5b0c20
```

`getFoo()` 函数中的逻辑看起来很奇怪，这里我们是故意为之，目的是让编译器无法进行返回值优化。有关返回值优化的介绍可查看[{{< ref "/post/cpp-copy-elision.md" >}}]({{< ref "/post/cpp-copy-elision.md" >}})。


结构 `Foo` 包含堆内存的成员数据，因此我们为其实现了一个深拷贝的复制构造函数（实际上还应重新实现赋值运算符）。通过运行输出可以看出来，在 `main` 函数中初始化变量 `f` 时是使用复制构造的：通过函数 `getFoo()` 返回的临时变量复制构造了 `f`。但仔细想一下，这里的堆内存深拷贝似乎没有什么必要，`getFoo` 函数会返回一个临时变量，然后通过这个临时变量拷贝构造了新的对象 `f`，但随后这个临时变量马上就被销毁了，这样一来一去似乎并没有什么意义，假如堆内存比较大，这样深拷贝的过程就会导致额外的性能损耗。

虽然我们可以使用常量左值引用的方式减少临时变量复制的开销，比如 `const Foo& f = getFoo();`，但这种方式并不能适用所有情况，因为它是常量的，意味着我们不能再对其进行任何的修改。

为了能避免这种类似的深拷贝消耗，C++11 引入了移动语义的概念。我们来看下面新的 C++11 代码是如何实现移动语义的：

```c++
#include <cstdio>
#include <cstring>

struct Foo
{
    char *data;

    Foo(const char* str) : data(new char[strlen(str) + 1])
    {
        strcpy(data, str);
        printf("constructor %p %p\n", this, data);
    }

    // 复制构造
    Foo(const Foo& other) : data(new char[strlen(other.data) + 1])
    {
        strcpy(data, other.data);
        printf("copy constructor %p %p\n", this, data);
    }

    // 移动构造
    Foo(Foo&& other) : data(other.data) 
    {
        other.data = 0;
        printf("move constructor %p %p\n", this, data);
    }

    ~Foo()
    {
        delete [] data;
        printf("destructor %p\n", this);
    }
};

Foo getFoo()
{
    if (true)
    {
        Foo a("blablabla");
        return a;
    }
    else
    {
        Foo b("foo");
        return b;
    }
}

int main()
{
    Foo f = getFoo();
    return 0;
}
```

相较于之前的例子，在这段代码中，我们为 `Foo` 增加了一个新的构造函数 `Foo(Foo&& other)`，它与复制构造函数不同的是，它接受一个右值引用类型的参数，在 C++11 中，这种接受一个右值引用类型的构造函数称之为**移动构造函数**（Move Constructor）。

根据前面有关右值的解释，在 `Foo f = getFoo();` 中，`getFoo()` 是一个右值，因此在初始化变量 `f` 时调用了接受右值引用参数的移动构造函数。可以看到，这个移动构造函数并没有像复制构造函数一样先分配内存再进行拷贝，它直接使用参数 `other` 的 `data` 成员初始化了本对象的 `data` 成员，并随后将 `other` 的 `data` 置为空值。

程序的运行输出如下：

```
constructor 0x7fff1be53190  0xf47010
move constructor 0x7fff1be531c0  0xf47010
destructor 0x7fff1be53190
destructor 0x7fff1be531c0
```

通过运行输出可以看到，整个过程中没有复制构造函数的调用，但有一次移动构造的过程，并且，最终变量 `f` 的 `data` 成员地址与临时变量的 `data` 成员的地址是相同的。通过这种移动构造的过程，将临时变量的数据直接交给了新的对象，临时变量不再持有原有的数据，从而避免了数据复制的过程，这种直接将数据“移为己用”的方式，就是所谓的移动语义。需要注意的是，在移动构造函数中，将参数 `other` 的 `data` 成员置为空是非常有必要的，否则在这个临时变量析构时，又将之前已经被移走的内存给释放了，导致新对象的 `data` 变为了一个悬挂指针，必然会导致严重的运行错误。

在这个例子中，如果我们不实现移动构造函数，而是只实现一个常规的接受常量左值引用的复制构造函数会发生什么？就像我们前面提到的，常量左值引用是个万能的引用，它既可以绑定左值，也可以绑定右值。如果 `Foo` 没有移动构造函数，`Foo f = getFoo();` 就会和 C++98 中一样，调用复制构造函数。这是一种非常安全的设计：如果移动不成，至少还可以进行拷贝。因此，在通常情况下，我们需要在实现移动构造函数的同时也实现常规的复制构造函数，以保证无法进行移动构造时，还可以使用复制构造。

除了移动构造之外，移动语义还包括移动赋值。下面是增加了拷贝赋值操作和移动赋值操作的代码：

```c++
#include <cstdio>
#include <cstring>

struct Foo
{
    char *data;

    Foo(const char* str) : data(new char[strlen(str) + 1])
    {
        strcpy(data, str);
        printf("constructor %p %p\n", this, data);
    }

    Foo(const Foo& other) : data(new char[strlen(other.data) + 1])
    {
        strcpy(data, other.data);
        printf("copy constructor %p %p\n", this, data);
    }

    Foo(Foo&& other) : data(other.data)
    {
        other.data = 0;
        printf("move constructor %p %p\n", this, data);
    }

    ~Foo()
    {
        delete [] data;
        printf("destructor %p\n", this);
    }

    // 拷贝赋值
    Foo& operator=(const Foo& other)
    {
        if (this != &other)
        {
            delete [] data;
            data = new char[strlen(other.data) + 1];
            strcpy(data, other.data);
        }
        printf("assignment %p %p\n", this, data);
        return *this;
    }

    // 移动赋值
    Foo& operator=(Foo&& other)
    {
        if (this != &other)
        {
            delete [] data;
            data = other.data;
            other.data = 0;
        }
        printf("move assignment %p %p\n", this, data);
        return *this;
    }
};

Foo getFoo()
{
    return Foo("blablabla");
}

int main()
{
    Foo f("1");
    f = getFoo();
    return 0;
}
```

拷贝赋值与移动赋值的关系，和拷贝构造与移动构造的关系类似。拷贝赋值通常接受一个常量左值引用类型的参数，并对数据进行深拷贝。而移动赋值接受一个右值引用类型的参数，并将右值临时量的数据移动以作为自身的数据。在 `f = getFoo();` 中，等号右侧是一个右值，因此执行了移动赋值的过程。

上面代码的执行输出如下：

```
constructor 0x7fff46c93fa0 0x1f2c010
constructor 0x7fff46c93fb0 0x1f2c030
move assignment 0x7fff46c93fa0 0x1f2c030
destructor 0x7fff46c93fb0
destructor 0x7fff46c93fa0
```

和移动构造的情况类似，如果我们没有实现移动赋值，只实现了拷贝赋值，`f = getFoo();` 就会调用拷贝赋值的过程，原因和刚才也是同样的，拷贝赋值的常量左值引用参数也可以绑定到右值。所以通常情况下，我们需要在实现移动赋值的同时也实现拷贝赋值的函数，以保证无法进行移动赋值时，还可以使用拷贝赋值。

通过以上这些例子我们可以看出，C++11 的移动语义可以将资源（堆内存，系统对象等）通过浅拷贝的方式从一个对象直接转移到另一个对象，减少不必要的数据拷贝，从而能够提高程序的性能，消除临时对象对性能的影响。右值引用最大的目的就是用来支持移动语义。如果你设计的某个数据结构包含像堆内存等需要拷贝的成员时，就可以通过移动语义来提高效率。当然，移动语义也不是必须的，如果数据复制的开销很小，也完全可以不使用移动语义。


### std::move

通过前面的介绍，我们了解了通过移动语义来避免不必要的数据拷贝，但实际情况往往比这些用于讲解的例子复杂，某些情况下很可能会出现明明想移动，却移不动，或者你以为移动了，实际却没移走的情况。来看下面这个例子：

```c++
#include <cstdio>
#include <cstring>

struct Bar
{
    Foo f; // 和前面例子中的Foo相同

    Bar(const char* str) : f(Foo(str)) { }
    Bar(const Bar& other) : f(other.f) { }
    Bar(Bar&& other) : f(other.f) { } // 移动构造
};

Bar getBar() 
{
    if (true) 
    {
        Bar a("blablabla");
        return a;
    }
    else 
    {
        Bar b("useless");
        return b;
    }
}

int main()
{
    Bar b2 = getBar();
    return 0;
}
```

结构 `Bar` 中包含一个 `Foo` 成员，这个就是我们前面例子中实现的 `Foo` 结构（包含了移动构造的实现），此处的代码中没有再重复列出。

结构 `Bar` 同样也实现了移动构造函数，由于 `Bar` 只包含一个 `Foo` 的成员，在移动构造 `Bar` 时，我们的目的就是将临时量 `other` 中的 `f` 成员移为己用。你可能会认为在这个移动构造函数中，因为参数 `other` 是一个右值临时量，所以 `other.f` 也是一个右值，所以 `f(other.f)` 调用的是 `Foo` 的移动构造函数，所以最终 `other.f.data` 被直接移到了 `this->f.data`，没有拷贝任何数据。

然而这却是一个巨大的误解，原因在于本文之前提到过的：一个变量自身是否为右值，和它的类型无关。

回想一下前面介绍的判断左值右值的方法：能够取地址的为左值，不能取地址的为右值。在 `Bar b2 = getBar();` 中，对等号右侧的表达式取地址 `&getBar()` 显然是不允许的，所以 `getBar()` 是一个右值。然而，在 `Bar` 的移动构造函数 `Bar(Bar&& other)` 之中，虽然参数 `other` 的类型为右值引用，但它本身却是一个左值，因为它此时已经具有名字 `other`，假如对其取地址：`&other` 也是没有问题的。同理，`other.f` 更是一个不折不扣的左值，所以 `f(other.f)` 调用的其实是 `Foo` 的复制构造函数，而不是移动构造函数。

但在这种情况下，`other` 代表的临时量确实马上就会被析构，`other.f` 也会随之消失，我们希望将它的资源也移动到新的变量中，该如何使其移动呢？解决方法是将 `other.f` 强行变为一个右值。C++11 已经考虑到了这种情况，提供了一个 `std::move` 函数（位于头文件 `<utility>`），它的功能就是将一个左值强制转换为右值引用，以用于移动语义。它基本上等同于:

```c++
static_cast<T&&>(lvaue);
```

所以，`Bar` 的移动构造函数应当这样实现：

```c++
Bar(Bar&& other) : f(std::move(other.f)) { } // 移动构造
```

需要注意，`std::move` 的函数名可能有一些迷惑性，这个函数本身并不能移动任何东西，它唯一的作用就是将一个左值强转为右值引用，需要搭配其它移动语义的实现才能真正的移动数据。在刚才的例子中，假如 `Foo` 并没有实现移动构造函数，`f(std::move(other.f))` 也无法移动数据，调用的还是复制构造函数。

## 通用引用

`T&&` 也不一定表示右值引用。前面我们提到过，通常情况下，右值引用不能绑定到一个左值。比如：

```c++
int a;
int && i = a; // 错误，a是一个左值
```

但在 `&&` 和类型推导结合时，它的类型是不定的，既可能是左值引用，也可能是右值引用。来看下面这个例子：

```c++
#include <iostream>
#include <type_traits>

template <typename T>
void f(T&& i)
{
    std::cout << std::boolalpha
              << std::is_rvalue_reference<T&&>::value << " "
              << std::is_lvalue_reference<T&&>::value << " "
              << i << std::endl;
}

int main()
{
    f(32);
    int x = 64;
    f(x);
    return 0;
}
```

C++11 新提供了 `std::is_lvalue_reference` 和 `std::is_rvalue_reference`（位于头文件 `<type_traits>`中），用于判断某个类型是左值引用还是右值引用。在这个例子中，我们定义了一个函数模板 `f`，它接受一个 `T&&` 类型的参数，在函数模板内，我们判断并输出这里的 `T&&` 是左值引用还是右值引用。这个例子的执行输出如下：

```
true false 32
false true 64
```

可以看到，执行 `f(32)` 时，`T&&` 是一个右值引用，而执行 `f(x)` 时，`T&&` 又是一个左值引用。

这是由于当 `T&&` 中的 `T` 需要进行类型推导时（比如在这里的函数模板中），`T&&` 不再表示一个右值引用，`T&&`既可以绑定到右值，也可以绑定到左值，这种结果类型不确定的引用，也称为通用引用（Universal Reference）。

Universal Reference并不是一个 C++ 标准中的称呼，这个名字来自 Scott Meyers 大神的总结，另外一些地方有时也称之为转发引用（Forwarding Reference）。

通用引用也是引用，所以它们必须被初始化。⼀个通用引用的初始值决定了它是代表了右值引用还是左值引用。如果初始值是⼀个右值，那么通用引用就会是对应的右值引用，如果初始值是⼀个左值，那么通用引用就是⼀个左值引用。对函数参数中的通用引用来说，初始值在调用函数的时候被提供。

需要注意，只有当 `T&&` 中的 `T` 需要进行类型推导时，`T&&` 才是一个通用引用（Universal Reference）。比如：

```c++
template <typename T>
void f(T&& i);            // T需要类型推导，T&&是universal reference

void f(int&& param);      // 没有类型推导，int&&是右值引用

template <typename T>
class Widget {
    ...
    Widget(Widget&& rhs); // 这里Widget是确定的类型，没有类型推导
    ...                   // Widget&&是右值引用
};

template<typename T1>
class Gadget {
    ...
    template<typename T2>
    Gadget(T2&& rhs);     // T2需要类型推导
    ...                   // T2&&为universal reference
};
```

对⼀个通用引用而⾔，类型推导是必要的，但只有类型推导还不够。声明引用的格式必须正确，必须是严格的 `T&&`。比如下面这个例子：

```c++
template<typename T>
void f(std::vector<T>&& param); // param是右值引用
```

这里看似既有类型推导，也有 `&&`，但并没有通用引用，因为这里的参数并不是 `T&&`，而是 `std::vector<T>&&`，所以即使看起来有类型推导，这里仍然是一个右值引用。

甚至一个简单的 const 修饰也会有影响，比如：

```c++
template <typename T>
void f(const T&& i);
```

这里的参数也不是通用引用，因为它在 `T&&` 之外又添加了 cv 限定。

有时在一个模板里见到了一个函数参数类型为 `T&&` 的形式，也未必是通用引用。比如 `std::vector` 的 `push_back` 函数：

```c++
emplate <class T, class Allocator = allocator<T> >
class vector {
public:
    ...
    void push_back(T&& x); // 右值引用
    ...
};
```

这里的 `T` 看似有类型推导，但其实不然。因为 `push_back` 作为一个成员函数，当它被调用时，`T` 的类型已经随 `vector` 的对象实例确定了，因此这个 `push_back` 中的 `T` 并没有发生任何类型推导，所以 `x` 属于右值引用，不属于通用引用。

除了模板类型推导，通用引用还可能存在 `auto` 声明中。

```c++
int i = 0;
auto&& a = i;
auto&& b = 10;
```

这里的 `a` 和 `b` 就属于通用引用，其中 `a` 由一个左值初始化，所以最终它的类型是左值引用，`b` 由一个右值初始化，所以最终它的类型是右值引用。

总结一下：

- 如果一个函数模板参数类型是 `T&&`，其中 `T` 需要进行类型推导得知，或者一个对象是用 `auto&&` 声明的，这个参数或者对象就是一个通用引用。
- 如果类型声明的形式不是准确的 `T&&`，或者没有发生类型推导，那么 `T&&` 代表一个右值引用。
- 当通用引用被一个右值初始化时，就会成为对应的右值引用；如果被一个左值初始化时，就会成为对应的左值引用。

## 完美转发

所谓完美转发，是指在函数模板中，完全依照模板参数的类型，将参数传递给函数模板中调用的另一个函数。这个要求看似非常简单但其实并不容易，来看下面这个例子：

```c++
void handle(int& i)
{
    std::cout << "lvalue ref" << std::endl;
}

void handle(int&& i)
{
    std::cout << "rvalue ref" << std::endl;
}

template <typename T>
void process(T&& x)
{
    handle(x);
}

int main()
{
    int a = 0;
    process(a); // 输出lvalue ref
    process(1); // 输出lvalue ref
}
```

通过上一部分的介绍可以看出，函数模板 `process` 的参数是一个通用引用，它既可以接受左值也可以接受右值，在这里我们本希望 `process` 能够将参数按照原本的的类型传递给 `handle` 函数：如果 `x` 是一个左值引用，则调用 `handle(int& i)`，如果 `x` 是一个右值引用，则调用 `handle(int&& i)`。但这段代码其实并不能如我们所愿，因为在 `process` 中 `x` 已经具名，它是一个左值，所以 `handle(x)` 始终调用的是 `handle(int& i)`。

可能会有人立马想到了前面介绍的 `std::move` 和 `std::is_rvalue_reference`，然后实现了一个这样的版本：

```c++
template <typename T>
void process(T&& x)
{
    if (std::is_rvalue_reference<T&&>::value)
        handle(std::move(x));
    else
        handle(x);
}
```

这样确实可行，但 C++11 给我们提供了更加优雅简洁的方法：使用 `std::forward`。

```c++
template <typename T>
void process(T&& x)
{
    handle(std::forward<T>(x));
}
```

`std::forward` 的作用也非常简单：当 `x` 的类型是左值引用时，返回左值引用，当 `x` 的类型是右值引用时，返回右值引用。

所以通过 `std::forward`，我们就能实现所谓的完美转发。

---

参考：

- 《深入理解C++11：C++11新特性解析与应用》机械工业出版社
- https://isocpp.org/blog/2012/11/universal-references-in-c11-scott-meyers
