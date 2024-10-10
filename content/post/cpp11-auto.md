---
title: 'C++11：auto 类型推导'
slug: cpp11-auto
date: 2016-06-20
tags: [c++]
---

C++ 是“静态类型”的语言，变量在定义的时候必须明确指定类型，而与之相对的“动态类型”语言，如 Python，JavaScript 等，则不需要指定变量的类型。C++11 引入了 `auto` 关键字来支持变量的自动类型推导，可以编写更简洁的代码，使开发者不必过于关注便变量的类型。

需要注意，这种自动类型推导是在编译时确定的，并不是真正的运行时动态类型，因此 C++ 仍然属于“静态类型”的。

## 基本用法

来看 auto 的一些基本用法：

```c++
int main()
{
    auto i = 1;     // i为int
    auto j = i + 1; // j为int
    auto d = 3.14;  // d为double
    auto p = &i;    // p为int*

    auto int m = 0; // error，不能同时指定auto和其他类型
    auto n;         // error，没有初始化，无法推导
}
```

可以看出，auto 是根据等号右侧的类型，也就是变量被初始化值的类型推导的。所以，通过 auto 声明变量时，必须同时进行初始化，才能让编译器推断出它的类型。auto 关键字只是一个变量类型声明的“占位符”，它的作用就是告诉编译器，让其在编译时自行推断出变量的类型。

## auto的优点

使用 auto 最明显的一个好处是可以让复杂类型变量的声明变得非常简单。

比如，以前要使用模板容器的迭代器，往往要写一长串的声明，而通过 auto 自动推导，可以使代码变得非常简洁易读：

```c++
std::vector<std::string> names;
......

// 以前的写法，冗长繁琐
std::vector<std::string>::iterator it = names.begin();
for (; it < name.end(); it++)
   ......

// 使用auto类型推导，简洁优雅
for (auto it = names.begin(); it < name.end(); it++)
   ......
```

除此之外，在一些不方便获取变量类型的地方，通过使用 auto 也可以让代码变得更简单，比如在一些模板的实现中：

```c++
template <typename Product, typename Worker>
void foo(Worker w)
{
    Product obj = w.produce();
    // do something about obj
}

class HatMaker
{
public:
    Hat produce();
};

int main
{
    HatMaker m;
    foo<Hat>(m); // 需指定函数模板参数Product
}
```

因为在函数模板`foo`中，不知道`produce`的返回类型，所以需要一个名为`Product`的模板参数，并且在调用函数模板时指定它的类型。但是通过 auto，在函数模板的实现中，就可以让其“自适应”`produce()`的返回类型：

```c++
template <typename Worker>
void foo(Worker w)
{
    auto obj = w.produce();
    // do something about obj
}

class HatMaker
{
public:
    Hat produce();
};

int main
{
    HatMaker m;
    foo(m);
}
```

## auto推导规则

auto 本质上是一个占位符，编译器在编译代码的时候将其替换为合适的类型声明。这种推导结果基本符合我们的直观想象，但有几点需要注意。

1. auto 不会主动推导为引用，除非 auto 和 `&` 一起结合使用。
2. 当 auto 推导结果不是引用或指针时，结果变量不会保留初始化表达式的 cv 属性。
3. 当 auto 推导结果是引用或者指针时，结果只保留被引用或被指向变量的 cv 属性。
4. auto 和 cv 限定符结合使用时，等同于为推导结果再加上相应的 cv 限定。

先来看第1条。例如，

```c++
int x = 1;
int &r = x;
auto a = x;   // a为int
auto b = r;   // b为int
auto & c = x; // c为int&
auto & d = r; // d为int&
```

虽然 r 是 `int&` 类型的，但是 b 并没有被推导为 `int&`，而 c 和 d 都是 `int&` 类型的，原因是 c 和 d 的初始化中， `&` 明确表示了它们是引用。

再来看第2，3条。看下面这个例子：

```c++
int x = 1;
const int cx = 1;

auto a = cx;   // a为int
auto &b = cx;  // b为const int&

int * p = &x;               // pointer to int
const int * pc = &x;        // pointer to const int
int * const cp = &x;        // const pointer to int
const int * const cpc = &x; // const pointer to const int

auto c = p;  // c为int*
auto d = pc; // d为const int*
auto e = cp; // e为int*
auto f = cpc;// f为const int*
```

可以看到，a 的推导结果为 int，并没有保留 cx 的 const 属性，但 b 的类型为 `const int&`，因为 b 被明确声明了是引用，所以也只能是 `const int&` 类型，保留了被引用对象 cx 的 const 属性。

c, d, e, f的情况就有点意思了，首先它们肯定都会被推导为指针，c 的结果肯定没有任何疑问。d 的结果为 `const int*`，因为 pc 的类型是指向 `const int` 的指针，所以保留了被指向类型的 const属性。而根据 e 和 f 可以发现，指针本身的 const 属性不能被保留，只能保留被指向类型的 const 属性。