---
title: '浅谈C++中的复制省略与返回值优化'
slug: cpp-copy-elision
date: 2017-12-20
tags: [c++]
---

在 C++ 中，复制省略指的是一种编译器优化，编译器采取优化手段来避免一些不必要的对象复制构造。返回值优化是复制省略优化中常见的一种情景之一。

## 复制省略

复制省略允许编译器省略掉一些不必要的对象复制构造过程。其中一个最常见的情况是用临时变量初始化对象变量的时候。在 C++ 中，声明并初始化一个对象一般有两种写法，比如：

```c++
std::string A("this is a");
std::string B = std::string("this is b");
```

请问这两种写法有什么区别？

通常情况下没有区别，两种写法的效果和效率是完全一样的，但这是因为编译器默认为我们做了优化。实际上这两种写法并不相同，前一种叫做直接初始化（Direct Initialization），它直接调用了 string 的构造函数来初始化 A 。后一种叫做复制初始化（Copy Initialization），在这句代码中，先调用 string 的构造函数创建了一个临时的 string 对象，也就是等号右侧的对象，然后再调用 string 的复制构造函数来创建对象 B，最后析构掉临时对象。但一般情况下，编译器优化会将这个复制的过程省略，因此，初始化 B 的过程和初始化 A 的过程是相同的，并没有调用 string 的复制构造函数。之所以有这种优化，是因为多出来的一次复制过程没有什么实际意义，如果某个类包含大量的数据，额外的复制过程显然会降低程序的性能。在 C++ 中，这种省略掉不必要的对象复制过程的优化就是**复制省略**（Copy Elision）。

需要注意，虽然在经过复制省略优化后，上面两句代码的效果效率相同，但在语法语义上，它们仍然属于不同的东西。第二种的复制初始化的写法仍然需要有可用的复制构造函数，假如复制构造函数被设为 `private` 或 `delete`，编译器就会报错。

来看一个简单例子：

```c++
class Data
{
public:
    Data() { printf("constructor\n"); }
    Data(const Data& other) { printf("copy constructor\n"); }
};

int main()
{
    Data d = Data();
    return 0;
}
```

我们可以通过 GCC 来编译实验这段程序，GCC 提供了 `-fno-elide-constructors` 参数来禁用复制省略，所以可以分别使用以下两个方式来编译程序观察一下运行结果。

```
g++ -o test1 main.cpp
g++ -o test2 main.cpp -fno-elide-constructors
```

默认情况下程序运行只有一行“constructor”输出，禁用掉复制省略后，会多一行“copy constructor”输出。

*注意：如果你禁用复制省略后仍然只有一行输出，那很可能因为 GCC 版本比较高，采用了 C++17 标准，可以添加 `-std=c++11` 参数指定一下语言标准，后文会解释具体原因。*

## 返回值优化

另外一种常见的复制省略情况是函数返回对象的时候，例如：

```c++
class Data
{
public:
    Data() { printf("constructor\n"); }
    Data(const Data& other) { printf("copy constructor\n"); }
};

Data getTemp()
{
    return Data();
}

Data getLocal()
{
    Data D;
    return D;
}

int main()
{
    Data A = getTemp();
    Data B = getLocal();
    return 0;
}
```

请问：在这个例子中，初始化 A 和 B 时，Data 被构造了几次？复制构造了几次？

如果是在没有任何优化的情况下，初始化 A 和 B 的过程中，Data 都会被构造 1 次，复制构造 2 次。初始化 A 时，先构造 getTemp() 函数内的临时对象，然后通过这个临时对象复制构造初始化 A 等号右侧的临时对象，最后再通过这个对象复制构造 A。初始化 B 时，先构造 getLocal() 函数内的对象 D，然后通过 D 复制构造初始化 B 等号右侧的临时对象，最后再通过这个对象复制构造 B。

但其实这些的复制过程都没有什么必要。所以，默认情况下编译器优化会省略掉这些复制，初始化 A 和 B 时都会分别只执行一次 Data 的构造函数。如果在 getLocal() 中打印 D 的内存地址，在 main() 中打印 B 的内存地址，就会发现它们的地址是完全一样的。

对于类似 getTemp() 这种直接返回临时对象时的复制省略，一般称为**返回值优化**（Return Value Optimization, RVO）。对于类似 getLocal() 这种返回一个命名对象时的复制省略，一般称为**命名返回值优化**（Named Return Value Optimization, NRVO）

有了这种返回值优化，一般就不需要专门关心函数返回对象时的复制消耗了，因为编译器替你省略了复制的过程。但是需要注意，在某些情况下编译器无法进行返回值优化，最常见的情况就是：函数根据不同的判断条件返回不同的命名对象。例如，

```c++
Data getData(bool which)
{
    Data a;
    Data b;
    if (which)
        return a;
    else
        return b;
    // or return which ? a : b;
}
```

这种情况下仍旧会执行一次复制构造。

## 注意

有关复制省略需要注意一点：复制省略优化确实有可能会改变程序的执行结果。就比如前面的几个例子，优化前后的输出内容不同，没有优化时会多一些输出。C++ 标准允许编译器对代码做任何优化，前提是不改变结果程序的“可观察到的行为（observable behavior）”，这个规定一般称之为 The As-if Rule。但复制省略优化是个例外，标准明确规定了允许这种省略，即使可能因为复制构造函数具有副作用而改变了程序的行为。所以，任何时候我们的代码逻辑都不应该依赖某个复制构造函数的副作用，一方面因为编译器会进行复制省略优化，另一方面这种逻辑会导致代码可读性很差，让人难以理解。

## C++17的改变

C++17 标准对复制省略做出了一些更新。

在 C++17 中规定，有两种情况不再属于优化操作，编译器必须进行复制省略。

一种就是类似于前面例子中 `Data d = Data();` 的情况，通过相同类型的临时变量初始化对象的时候。另一种就是类似于前面 `getTemp()` 的情况，函数直接返回临时对象的时候。

这也就是为什么前面提到的，如果使用比较新的 GCC 且默认标准为 C++17 的话，即使加上 `-fno-elide-constructors` 参数，也没有复制构造函数输出的原因。

---

参考：

- https://en.cppreference.com/w/cpp/language/copy_elision
- https://en.cppreference.com/w/cpp/language/as_if
- https://en.wikipedia.org/wiki/Copy_elision
- http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0135r0.html
