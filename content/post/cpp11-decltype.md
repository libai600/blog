---
title: 'C++11：decltype 类型推导'
slug: cpp11-decltype
date: 2016-06-29
tags: [C++]
---

除了 `auto` 以外，C++11 中新增了 `decltype` 关键字，也用于自动类型推导，它和 `auto` 推导类似，但又有明显的区别。

## 基本使用

先来看下面的例子。

```c++
int a = 1;
decltype(a) b = a; // b: int
decltype(a) c;     // c: int
decltype(a + c) d = 1.2; // d: int
decltype(a + 1.0) e = 1; // e: double
```

decltype 的语法格式为 `decltype(exp)`，它根据其中表达式 `exp` 的类型进行推导。

在这个例子中，b 的类型由 `decltype(a)` 推导确定，而由于 a 的类型为 int，所以 b 类型也为 int。同理，c 的类型也为 int。d 的类型根据表达式 a + c 的类型确定，同样为 int（与等号右侧1.2的类型无关）。在 e 的声明中，因为 1.0 默认为 double，所以表达式 a + 1.0 的类型为 double，e 的类型也为 double。

`decltype(exp)` 的这种类型推导和`auto` 类似，都是在编译时期完成的，所以并不会真正计算表达式 `exp` 的值，它只是需要判断 `exp` 的类型。

对比一下 `decltype` 和 `auto` ：

```c++
auto a = value;
decltype(exp) b = value;
```

它们的不同点在于，auto 是根据变量的初始化表达式来推导出最终类型，也就是等号右侧的类型，而 decltype 是根据其中 exp 表达式的类型进行推导，与等号右侧的值类型无关。因此，在使用 auto 声明的变量时也必须同时进行初始化，而使用 decltype 声明变量时不一定必须初始化，就比如最上面例子中的 `decltype(a) c;`。（实际上使用 decltype 声明变量时是否需要初始化，与推导的类型有关，后面会进行解释）


## 推导规则

多数时候 decltype 的推导结果都比较符合程序员的直觉想象，但有些时候还是会和预期猜想不同，导致一些奇怪的错误，因为 decltype 的推导规则还是有一些复杂的。

对于 `decltype(e)` ，理论上 `e` 可以是任意类型的表达式，其推导按如下规则依次进行：

1. 如果 e 是一个没有带括号的标记符表达式（id-expression）或者类成员访问表达式，则 `decltype(e)` 就是 e 的类型。此外，如果 e 是一个被重载的函数，则会导致编译错误；
2. 否则，假设 e 的类型为 T，如果 e 是一个将亡值（xvalue），则 `decltype(e)` 类型为`T&&`；
3. 否则，假设 e 的类型为 T，如果 e 是一个左值（lvalue），则`decltype(e)` 类型为`T&`；
4. 否则，假设 e 的类型为 T，则 `decltype(e)` 类型为 `T` 。

这里解释一下标记符表达式（id-expression），基本上，所有除关键字，字面量，等编译器需要使用的标记之外的程序员自定义的标记，都可以是标记符（identifier），而单个标记符对应的表达式就是标记符表达式。比如，`int arr[4]` ，那么 `arr` 是一个标记符表达式，而 `arr[3] + 0` , `arr[3]` 等，都不是标记符表达式。

这个规则看起来非常复杂，但实际应用中，最容易引起迷惑的只有规则 1 和 3。这里我们通过一些例子来理解一下。

```c++
int i = 4;
int arr[5] = {0};
int *ptr = arr;

struct S { double d; } s;

void Overloaded(int);
void Overloaded(char); // 重载的函数

int && RvalRef();

const bool Func(int);

// 规则1：单个标记符表达式一起访问类成员，推导为本类型
decltype(arr) var1;              // int[5]，标记符表达式
decltype(ptr) var2;              // int*，标记符表达式
decltype(s.d) var4;              // double，成员访问表达式
decltype(Overloaded) var5;       // 错误，是个重载的函数

// 规则2：将亡值，推导为类型的右值引用
decltype(RvalRef()) var6 = 1;    // int&&

// 规则3：左值，推导为类型的引用
decltype(true ? i : i) var7 = i; // int&，三元运算符，这里返回一个i的左值
decltype((i)) var8 = i;          // int&，带圆括号的左值
decltype(++i) var9 = i;          // int&，++i返回i的左值
decltype(arr[3]) var10 = i;      // int&，[]操作符返回左值
decltype(*ptr) var11 = i;        // int&，*操作符返回左值
decltype("lval") var12 = "lval"; // cosnt char(&)[9]，字符串字面常量为左值

// 规则4：以上都不是，推导为本类型
decltype(1) var13;               // int，除字符串外字面常量为右值
decltype(i++) var14;             // int，i++返回右值
decltype((Func(1))) var15;       // const bool，圆括号可以忽略
```

以上代码将四个规则的例子都列了出来。可以看到，规则 1 不但适用于基本数据类型，还适用于指针，数组，结构体，甚至函数类型的推导，实际上规则 1 在 `decltype` 类型推导中运用最为广泛。而规则 2 也比较简单，基本符合程序员的想象。

规则 3 其实是一个左值规则。decltype 的参数不是标记符表达式或者类成员访问表达式，且参数都为左值，推导出的类型均为左值引用。这里有一点比较特别，需要额外注意，`(i)` 不属于标记符表达式，而是一个左值表达式，因此 `decltype((i))` 和 `decltype(i)` 的类型不同。规则 4 则是用于以上都不适用者。我们这里看到了 `++i` 和 `i++` 在左右值上的区别，以及字符串字面常量 `lval` ，非字符串字面常量 1 在左右值间的区别。

decltype 类型推导也会保留原类型的 cv 属性，参考下面这个例子：

```c++
int i = 1;                  // int
int &ri = i;                // reference to int
const int ci = 1;           // const int
const int &cri = i;         // const reference to int
int *pi = &i;               // pointer to int
const int *pci = &i;        // pointer to const int
int* const cpi = &i;        // const pointer to int
const int* const cpci = &i; // const pointer to const int

decltype(i) a;              // int
decltype(ri) b = a;         // int&
decltype(ci) c = a;         // const int
decltype(cri) d = a;        // const int&
decltype(pi) e = &a;        // int*
decltype(pci) f = &a;       // const int*
decltype(cpi) g = &a;       // int* const
decltype(cpci) h = &a;      // const int* const

std::cout << std::is_same<decltype(i), int>;::value << std::endl;
std::cout << std::is_same<decltype(ri), int&>::value << std::endl;
std::cout << std::is_same<decltype(ci), const int>::value << std::endl;
std::cout << std::is_same<decltype(cri), const int&>::value << std::endl;
std::cout << std::is_same<decltype(pi), int*>::value << std::endl;
std::cout << std::is_same<decltype(pci), const int*>::value << std::endl;
std::cout << std::is_same<decltype(cpi), int* const>::value << std::endl;
std::cout << std::is_same<decltype(cpci), const int* const>::value << std::endl;
```

在这个例子中，最终变量 `a` 到 `h` 的类型，分别与声明它们时 `decltype` 中变量的类型完全一样。在调试程序时，调试器可能无法精确的显示变量的 cv 属性，所以在这里我们借助 `std::is_same` 来验证类型是否如我们预期，如果类型 `T` 和类型 `U` 相同，则 `std::is_same<T,U>::value` 的值为 `true` 。在这个例子中，最终的输出为八个1，也就表明了 `decltype` 推导能够保留原来的 cv 属性。

---

参考：

- 《深入理解C++11：C++11新特性解析与应用》机械工业出版社
- https://en.cppreference.com/w/cpp/language/decltype
