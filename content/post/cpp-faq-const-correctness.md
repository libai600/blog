---
title: '[译] C++ 常见问题之：Const 限定'
slug: cpp-faq-const-correctness
date: 2015-05-10
tags: [C++]
---

> 这篇是 [isocpp.org](http://isocpp.org) 上 [C++ Super-FAQ](https://isocpp.org/faq) 系列中的一篇。包含了 C++ 中有关 const 关键字的一些常见问题。发现这篇文章作者写的比较有意思，故做翻译交流，水平有限，欢迎指正。  
> 原文：[https://isocpp.org/wiki/faq/const-correctness](https://isocpp.org/wiki/faq/const-correctness)

## 什么是"const限定"？

_What is "const correctness"?_

好东西。意思是使用 **const** 关键字来防止对象发生变化。 

举例来说，假如你要创建一个函数 `f()`，它接收一个 `std::string` 类型的参数，而且保证不修改传入的 `std::string` 参数对象，那可以让 `f()` 这样接收参数：

```c++
void f1(const std::string& s);      // Pass by reference-to-const
void f2(const std::string* sptr);   // Pass by pointer-to-const
void f3(std::string s);             // Pass by value
```

对于 `f1()` 和 `f2()` ，在函数中，任何试图修改这个 `std::string` 的代码都会导致编译错误。因为这种检查是在编译过程中完成的，所以 `const` 不会导致任何额外的运行时开销。在 `f3()` 中，函数获得了一份 `std::string` 的拷贝，`f3()` 可以任意修改这份拷贝，并且拷贝的对象会在 `f3()` 返回时被析构，`f3()` 也无法修改调用者的那份 `std::string` 对象。 

再看一个相对应的例子，假如这回你要创建一个函数 `g()`，它接收一个 `std::string`，但是想让调用者知道函数 `g()` 可能会修改调用者的 `std::string` 对象。这种情况下可以让函数 `g()` 这样接收参数：

```c++
void g1(std::string& s);      // Pass by reference-to-non-const
void g2(std::string* sptr);   // Pass by pointer-to-non-const
```

函数中没有了 `const`，相当于告诉编译器，它们可以修改调用者的 `std::string` 对象了（是可以修改，不是必须修改）。所以，可以将 `std::string` 对象传递给前面任何一个 `f()` 函数，但只有 `f3()`，（就是通过值传递的那个）才能将这个 `std::string` 再传给 `g1()` 或者 `g2()`。如果 `f1()` 和 `f2()` 想调用任何一个 `g()` 函数，必须在函数内部复制一份再传给 `g()`。`f1()` 和 `f2()` 的参数不能直接传给 `g()`。比如：

```c++
void g1(std::string& s);
void f1(const std::string& s)
{
    g1(s);          // Compile-time Error since s is const
    std::string localCopy = s;
    g1(localCopy);  // Okay since localCopy is not const
}
```

在上面的例子中，`g1()` 是对 `f1()` 中的 `localCopy` 对象做修改，而通过 `const` 引用传给 `f1()` 的对象则不会被修改。

## const 限定和一般的类型安全如何联系？

_How is "const correctness" related to ordinary type safety?_

将形参声明为 `const` 就是另一种形式的类型安全。

如果你发现一般的类型安全能帮你的系统正确运行（确实能，尤其是大型系统），那 `const` 限定同样也能。

`const` 限定的好处是它能防止你无意中修改了本来不想修改的东西。你只不过需要再多敲几个字母来装饰一下代码，但是好处是编译器和其他程序员获得了一些额外的重要信息，从而编译器能阻止一些错误，程序员也能更好的理解代码。 

从概念上来说，你可以认为 `const std::string` 和普通的 `std::string` 是不同的类型，因为 non-const 类型上那些可以做修改的操作在 `const` 类型上都没了。就比如，可以认为 `const std::string` 只是没有了赋值操作符 `+=` 以及其他能进行修改的操作。

## 我是应该在代码中提早加上 const，还是等后面再补？

_Should I try to get things const correct "sooner" or "later"?_

在最最最最开始的时候加上。

回头再补 `const` 限定的话，会导致滚雪球效应：代码里面在一个地方想补个 `const`，就得在其它地方补好几个 `const`。

所以，尽早加上。

## "const X* p" 是什么意思？

_What does "const X* p" mean?_

意思是，指针 `p` 指向一个 `X` 类型的对象，但是不能通过 `p` 来修改这个对象（当然 `p` 也可以是空指针）。

从右往左读：`p` 是一个指针，它指向一个 `X` 对象，这对象是 `const` 的。

*译注：原文为“Read it right-to-left: p is a pointer to an X that is constant.”，英语更好理解。* 

如果 `X` 有个 const 成员函数 `inspect() const`，`p->inspect()` 这样是可以的。但是如果 X 有个 non-const 成员函数 `mutate()`，`p->mutate()` 这样就是错误的。

值得注意的是，这个错误是编译器在编译的时候发现的，没有运行时的检测。所以也意味着 `const` 不会拖慢你的程序，也不需要你写额外的运行测试案例，编译器在编译过程中就搞定了。

## "const X* p","X* const p"和"const X* const p"有什么区别？

_What's the difference between "const X* p", "X* const p" and "const X* const p"?_

指针的声明要从右往左读。

-   `const X* p` 表示 `p` 是一个指针，指向一个 `X` 类型对象，这对象是 const 的。通过 `p` 无法修改这个 `X` 类型对象。
-   `X* const p` 表示 `p` 是一个 const 指针，它指向一个 `X` 类型对象。指针 `p` 本身是不能修改的，但是能通过 `p` 修改这个 X 对象。
-   `const X* const p` 表示 `p` 是一个 const 指针，它指向一个 `X` 类型对象，这对象也是 const 的。不能修改 `p` 本身，也不能通过 `p` 修改这个 `X` 对象。

噢耶，我之前说过指针要从右往左读吧？

> *译注：在英文中，指针从右往左读是非常自然的，此处附上原文以助理解：*
> 
> -   `const X* p` means "p points to an X that is const": the X object can't be changed via p.
> -   `X* const p` means "p is a const pointer to an X that is non-const": you can't change the pointer p itself, but you can change the X object via p.
> -   `const X* const p` means "p is a const pointer to an X that is const": you can't change the pointer p itself, nor can you change the X object via p.

## "const X& x"是什么意思？

_What does "const X& x" mean?_

意思是，`x` 是一个 `X` 对象的别名，但是你不能通过 `x` 修改这个 `X` 对象。

从右往左读：`x` 是一个引用，引用的是一个 `X` 类型对象，这个对象是 const 的。 *原文：“x is a reference to an X that is const.”* 

如果 `X` 有个 const 成员函数 `inspect() const`，`x.inspect()` 这样是可以的。但是如果 `X` 有个 non-const 成员函数 `mutate()`，`x.mutate()` 这样就是错误的。 

这就和 pointer-to-const 是一个效果，当然也包括了前面提到的，编译器在编译过程中完成了所有的检查，const 不会拖慢你的程序，也不需要你写额外的运行测试案例。

## "X const& x"和"X const* p"是什么意思？

_What do "X const& x" and "X const* p" mean?_

`X const& x` 等同于 `const X& x`，`X const* p` 等同于 `const X* p`。

有些人喜欢把 const 放在右边，这种风格的写法称之为 "consistent const"，或者使用 Simon Brand 创造的术语 "East const" 来表示。"East const" 的写法确实比其他写法看起来更“一致”：这种写法始终把 const 关键字放在它要限定的东西的右边，而其他的风格有时放左边有时放右边。

根据 "East const" 写法，在定义 const 局部变量的时候要把 const 放在右侧：`int const a = 42;`，类似的，静态 const 变量这样定义：`static double const x = 3.14;`。基本上 const 左边的东西就是它要限定的部分，比如 const 指针的声明，const 成员函数等等。 

在使用类型别名的时候，"East const" 写法也不容易搞混：比如在下面的代码中，`foo` 和 `bar` 是两种不同的类型，为什么？

```c++
using X_ptr = X*;
const X_ptr foo;
const X* bar;
```

换成 "East const" 写法就清楚了：

```c++
using X_ptr = X*;
X_ptr const foo;
X* const foobar;
X const* bar;
```

这就比较明显了，`foo` 和 `foobar` 是相同的类型，而 `bar` 是另外的类型。

另外，在声明指针的时候，"East const" 写法会更清楚容易理解。对比一下传统的写法：

```c++
const X** foo;
const X* const* bar;
const X* const* const baz;
```

和 "East const" 写法：

```c++
X const** foo;
X const* const* bar;
X const* const* const baz;
```

虽然有这些好处，但是 const 放右边的写法不怎么流行，旧工程更倾向于用传统的写法。

## "X& const x"有意义吗？

_Does "X& const x" make any sense?_

没有，这么写完全没意义。

要弄明白它表示什么，还得从右往左读：x 是一个 const 的引用，指向一个 X 对象（x is a const reference to a X）。但这样是多余的，引用始终都是 const 的，你绝不可能重新设置一个引用，让他指向另外一个对象，跟有没有 const 没关系。

也就是说，`X& const x` 在功能上等同于 `X& x`。既然在 & 后面加个 const 没有任何作用，那就不该加它。加了反而容易让人产生误解，误以为像 `const X& x` 中的 X 一样是 const 的。

## 什么是“const成员函数”？

_What is a "const member function"?_

一种只访问但不修改其内部对象的成员函数。

把一个 const 后缀加到成员函数参数列表最后面，表示它是一个 const 成员函数。带有 const 后缀的成员函数就叫“const成员函数(const member functions)”，或者 “inspectors”。没有 const 后缀的成员函数叫“non-const成员函数(non-const member functions)”，或者 “mutators”。 *译注：此处提到了一个 inspector 函数和 mutator 函数的概念，后续内容会经常提到这两个概念。*

```c++
class Fred {
public:
    void inspect() const;   // This member promises NOT to change *this
    void mutate();          // This member function might change *this
};
void userCode(Fred& changeable, const Fred& unchangeable)
{
    changeable.inspect();   // Okay: doesn't change a changeable object
    changeable.mutate();    // Okay: changes a changeable object
    unchangeable.inspect(); // Okay: doesn't change an unchangeable object
    unchangeable.mutate();  // ERROR: attempt to change unchangeable object
}
```

试图调用 `unchangeable.mutate()` 会导致编译错误。这里的 const 不会有运行时间和内存上的消耗，你也不需要编写运行时的测试案例。

inspect() 函数后面的 const 应当用来表示，这个函数不会改变对象抽象层面的（用户视角的）状态。这种说法和“不会更改这个对象结构的二进制bit”存在区别。C++ 编译器不可能采用“逐位”的解释，除非他们能解决别名的问题，正常来说这问题不可能被解决（比如，有可能存在一个 non-const 的别名，通过它可以修改对象的状态）。另外一个有关别名问题的观点是：通过 pointer-to-const 指向一个对象，并不能保证这个对象不会变化，它只不过是能保证无法通过这个指针修改这个对象。 *译注：此处的“别名”指的是对象的指针和引用，一个对象的指针或者引用可以看作是它的别名，因为逻辑上它们表示的都是同一个对象。*

## 返回引用和 const 成员函数之间有什么联系？

_What is the relationship between a return-by-reference and a const member function?_

在 inspector 函数中，如果想要返回这个对象的某个成员，应该通过 reference-to-const 返回，比如 `const X& inspect() const`，或者直接按值返回，比如 `X inspect() const`。

```c++
class Person {
public:
  const std::string& name_good() const;  // 正确：调用者无法修改Person的name
  std::string& name_evil() const;        // 错误: 调用者能够修改Person的name
  int age() const;                       // 正确: 调用者无法修改Person的age
  // ...
};
void myCode(const Person& p)  // myCode()本想保证不会修改这个Person对象...
{
  p.name_evil() = "Igor";     // 但是myCode()还是修改了!!
}
```

好消息是编译器通常能揪出你犯的这类错误。尤其是你不小心通过 non-const 引用返回了一个对象的成员变量，就比如上面的 `Person::name_evil()`，编译器通常能够检测出来，并且在编译到 `Person::name_evil()` 内时，报给你个编译错误信息。

坏消息是编译器不是每次都能检测到：有些情况下，编译器根本不会给你一个编译错误消息。

也就是说：你得自己思考，如果你害怕了，就换个别的工作。“思考”比骂人难。

请记住贯穿这一节的“const哲学”：const 成员函数不能够修改（也不允许调用者修改）这个对象的逻辑状态（也叫抽象状态，表示状态）。你需要想一下这对象表示的是什么，而不是它内部是怎么实现的。一个人的年龄和姓名在逻辑上是这个人的一部分，但是这个人的邻居和老板就不是。当 inspector 函数返回这个对象的一个逻辑组成部分时，不能返回这个部分的 non-const 指针或引用，不管这部分是不是通过对象内部的成员变量直接实现的，还是其他什么方式实现的。

## "const重载"是怎么一回事？

_What's the deal with "const-overloading"_

const 重载能帮助你实现 const 限定。

当你有个 inspector 函数和一个 mutator 函数，他俩函数名一样，参数类型和数量也一模一样的时候，就是 const 重载。这俩函数的唯一不同点是，inspector 是 const的，而 mutator 是 non-const 的。

const 重载最常见的用法是在下标操作符中。一般情况下，你应该尽量使用标准容器模板，比如 `std::vector`，但是如果你想自己实现一个带下标操作符的类，那么一个经验法则就是：下标操作符通常成对出现。

```c++
class Fred { /*...*/ };
class MyFredList {
public:
  const Fred& operator[] (unsigned index) const;  // 下标操作符通常成对出现
  Fred&       operator[] (unsigned index);        // 下标操作符通常成对出现
  // ...
};
```

这个 const 下标操作符返回一个 const 引用，所以编译器会阻止调用者有意无意的修改这个 Fred。non-const 下标操作符返回一个 non-const 引用，通过这种方式，你告诉了调用者（还有编译器）它们允许修改这个 Fred 对象。

当一个 MyFredList 类的使用者调用下标操作符时，编译器会根据他这个 MyFredList 对象的 const 限定，来选择调用哪个重载。如果调用者手里的是 `MyFredList a` 或者 `MyFredList& a`，那么 `a[3]` 会使用 non-const 的下标操作符，而且调用者最终得到一个 Fred 的 non-const 引用。

例如，假设 Fred 类有一个 inspector 函数 `inspect()`，和一个 mutator 函数 `mutate()`：

```c++
void f(MyFredList& a)  // The MyFredList is non-const
{
  // Okay to call methods that inspect (look but not mutate/change) the Fred at a[3]:
  Fred x = a[3];       // Doesn't change to the Fred at a[3]: merely makes a copy of that Fred
  a[3].inspect();      // Doesn't change to the Fred at a[3]: inspect() const is an inspector-method
  // Okay to call methods that DO change the Fred at a[3]:
  Fred y;
  a[3] = y;            // Changes the Fred at a[3]
  a[3].mutate();       // Changes the Fred at a[3]: mutate() is a mutator-method
}
```

但是如果调用者手里的是 `const MyFredList a` 或者 `const MyFredList& a`，那么 `a[3]` 会使用 const 的下标操作符，调用者最终得到一个 Fred 的 const 引用。这种情况允许调用者查看 `a[3]`，但是会阻止调用者修改 `a[3]` 上的那个 Fred 对象。

```c++
void f(const MyFredList& a)  // The MyFredList is const
{
  // Okay to call methods that DON'T change the Fred at a[3]:
  Fred x = a[3];
  a[3].inspect();
  // Compile-time error (fortunately!) if you try to mutate/change the Fred at a[3]:
  Fred y;
  a[3] = y;       // Fortunately(!) the compiler catches this error at compile-time
  a[3].mutate();  // Fortunately(!) the compiler catches this error at compile-time
}
```

有关下标以及其他操作符函数的 const 重载在[这个链接](https://isocpp.org/wiki/faq/operator-overloading#matrix-subscript-op)，[这个链接](https://isocpp.org/wiki/faq/freestore-mgmt#multidim-arrays2)，[这个链接](https://isocpp.org/wiki/faq/freestore-mgmt#multidim-arrays3)，[这个链接](https://isocpp.org/wiki/faq/freestore-mgmt#multidim-arrays4)还有[这个链接](https://isocpp.org/wiki/faq/templates#class-templates)进行了说明。 当然，除了下标操作符，你也可以在其他地方使用 const 重载。

## 区分出逻辑状态和物理状态，为什么能帮助我设计出更棒的类？

_How can it help me design better classes if I distinguish *logical state* from *physical state*?_

因为它能促使你从外到内的，而不是从内到外的设计类，让你的类和对象更容易理解，更方便使用、更直观、更不容易出错和更快。（好吧，这么说有点过于简化了。但是想要理解所有的前因后果，你得接着往下看完这部分！）

让我们从内到外地理解这一点 -- [您应该从外到内设计您的类](https://isocpp.org/wiki/faq/operator-overloading#design-interfaces-first)，但是如果你对这个概念不熟悉，从内到外理解它更容易点。

在内部，你的对象具有物理的（或者叫具象的，按位的）状态。对于程序员来说，很容易观察和理解这种状态。如果某个类（class）只是个 C 语言风格的结构体（C-style struct）的话，它就是这种状态。

在外部，你的对象有它的使用者，并且限制了这些使用者只能用 public 成员函数和友元。这些外部的使用者也认为对象具有状态，举个例子，假如有个 `class Rectangle` 类型的对象，带有函数 `width()`, `height()` 和 `area()`，那么用户就会认为它们都是这个对象逻辑状态（或者叫抽象的，意象的）的一部分。对于外部使用者来说，这个 Rectangle 对象的确有个 area 面积，即便这个面积是动态计算出来的（比如 `area()` 函数返回的这个对象宽度和高度的乘积）。实际上，用户不知道也不关心你是怎么实现这些函数的，这一点很重要。从用户的视角来看，他们就认为这个对象在逻辑层面上，有宽度，高度和面积三个状态。

这个面积的例子表明，有时候逻辑状态没有对应的物理状态。反过来也一样：有时候 class 会故意隐藏一些物理状态，故意不提供对应的 public 函数，从而用户就无法访问和修改，甚至都不知道有这个物理状态。这也就意味着，对象中有一些物理状态没有对应的逻辑状态。

这种情况有个例子，一个用于集合保存某些信息的对象，可能会缓存上一次检索的结果，用来提高下一次检索的性能。这个缓存数据肯定属于对象的物理状态，但也是内部的实现细节，不应该暴露给用户，所以，不属于对象的逻辑状态。如果你从外到内考虑，就比较容易区分：如果这个对象的用户没有办法查看缓存本身的状态，那这个缓存就是透明的，也不属于对象的逻辑状态。

## 成员函数的const限定应该基于对象的逻辑状态还是物理状态？

_Should the constness of my public member functions be based on what the method does to the object's logical state, or physical state?_

逻辑的。

*译注：可以将物理状态认为是物理内存状态，更好理解。* 

接下来的内容可不简单，而且会让你觉得痛苦，所以建议你最好还是坐好了，而且为了你自己的安全，请务必确保周围没有锋利的东西。

我们先回顾一下前面刚提到的[这个例子](#区分出逻辑状态和物理状态为什么能帮助我设计出更棒的类)。记住：这个检索的函数会缓存上一次的结果，用来加速以后的检索。

先声明一点：假设检索函数没有改变对象的任何逻辑状态。

那么，是时候让你感受痛苦了，准备好了吗？

听好了：如果这个检索函数没有改变对象的逻辑状态，但是改变了对象的物理状态（它确确实实修改了一个确实存在的缓存数据），那么这个检索函数是否应该为 const 的？ 

答案为响当当的：是。（凡事都有例外，所以这个“是”旁边该有个星号，但绝大多数情况下，答案为“是”）

这就是 “逻辑const” 高于 “物理const”。意思是说，是否使用 const 修饰函数，主要取决于该函数是否会保持逻辑状态不变，不管它...（你坐好了没？最好赶紧坐下）不管它是不是碰巧对真实物理状态进行了非常真实的修改。

要是你没明白，或者说是没感受到痛苦，让我们把它分成两种情况：

- 如果一个成员函数改变了对象的任意逻辑状态，那它就属于一个 mutator 函数，它不应当是 const 的，即使它没修改这个对象上的任何一个物理内存 bit（确实有可能！）。
- 相反，如果一个成员函数从来没修改过对象的任意逻辑状态，那它属于一个 inspector 函数，并且应该是 const 的，即使它更改了对象上的某个物理内存 bit（确实也有可能！）。

如果你晕了，那就再读一遍。

如果你没晕，但是很抓狂，很好。你可能不喜欢，但起码懂了。深呼吸，然后跟着我念：函数的 const 属性应当对对象外部有意义。

如果你还是很抓狂，重复念三遍：函数的 const 属性，一定是对使用者有意义，并且使用者只能看到对象的逻辑状态。

如果你仍然还是跟抓狂，那不好意思，它就是这样的，忍着慢慢去习惯。没错，会有例外，凡事都有例外，什么规则都有例外，但作为这么一条规则，一般情况下，这个“逻辑 const”的观点，对你和你的软件都有好处。

还一件事，讲这个有些空洞，但是让我们明确一个函数是否改变对象的逻辑状态。从一个类的外部角度看，假如你是一个普通的使用者，你可以执行的每个实验（调用的函数，或者一系列函数）都会得到相同的结果（相同的返回值，相同的异常或者都没有异常），不管是不是先调用了那个检索函数。如果这个检索函数改变了后来调用的任何函数的行为（不只是变快了，而且改变了输出，返回值，异常），那么这个检索函数就改变了对象的逻辑状态，所以它是一个 mutator 函数。但是如果这个检索函数除了让某些东西更快以外没改变任何东西，那它是一个 inspector。

## 如果我想让const成员函数“悄悄的”修改成员变量，该怎么办？

_What do I do if I want a const member function to make an "invisible" change to a data member?_

使用 `mutable`（或者使用 `const_cast` 作为最后的手段）。

一小部分 inspector 函数需要修改对象的物理状态，但又不能让外部用户察觉到，也就是对物理状态的更改，而不是逻辑状态。

比如前面讨论的信息收集对象(collection-object)，它会缓存上一次的检索结果，以改善下一次的检索性能。在这个例子中，既然通过 public 接口无法直接观察到这个缓存数据，那么这个缓存就不属于对象的逻辑状态，所以对它的修改是对外部用户不可见的。既然检索函数没有改变任何逻辑状态，那这个它就属于 inspector 函数，尽管它修改了对象的物理状态。

当一个函数改变了物理状态但是没改变逻辑状态，那它就应该被标记位 const，因为实际上它是一个 inspector 函数。这就导致了一个问题：当编译器发现你的 const 函数修改了对象的物理状态时，它会发牢骚，给你个错误信息。

C++ 编译器使用关键字`mutable`来帮助你拥抱这个逻辑 const 的理念。这本例中，你要使用`mutable`关键字来标记这个缓存数据，通过这种方式编译器就知道在，const 函数内，或者通过 const 指针/引用来修改缓存是允许的。用我们的行话来说，`mutable`关键字标记了物理状态中不属于逻辑状态的那一部分。

mutable 关键字位于成员变量声明的前面，和你可能放 const 关键字的地方一样。另外一种非首选的方式为，把 this 指针的 const 属性去掉，大概需要用`const_cast`：

```c++
Set* self = const_cast<Set*>(this);
  // 这么干之前先看看下面的说明！
```

这行代码之后，`self` 和 `this` 的值一摸一样，也就是 `self == this`，但是 `self` 是 `Set*` 类型，而不是 `const Set*` 类型（严格来说 `this` 是 `const Set* const` 类型，但是最右边的 const 跟这个议题无关）。这就意味着你可以通过 `self` 来修改 `this` 表示的对象了。

**注意**：使用 `const_cast` 可能产生一个非常罕见的错误。它只会在这三个事同时存在时发生：一个成员变量应当是 mutable 的（就像前面讨论的）；编译器不支持 `mutable` 关键字，或者程序员没使用 `mutable`；一个对象原本就是 const 常量（本来就是 const 的，不是用 const 指针指向一个非 const 的变量）。尽管这种组合非常罕见，可能你永远不会碰上，但是一旦发生，代码可能就不工作了（标准上说行为未定义）。 如果你想用 `const_cast`，请使用 `mutable` 代替。如果你想修改一个对象的成员变量，但是这个对象是通过 pointer-to-const 指向的，最安全和最简单的做法就是吧 mutable 加到成员变量声明的前面。要是你能确保这个对象原本不是 const 的（比如你确定这个对象是这样声明的：`Set s;`），那可以用 const_cast；但是如果这个对象可能是 const 的（比如是这样声明的：`const Set s;`），那就使用 `mutable` 而不是 `const_cast`。 请不要说什么，X 平台上的 Y 版本的 Z 编译器允许你修改 const 对象的 non-mutable 成员。我不管那个，根据语言标准这是非法的操作，换个编译器甚至换个相同编译器的不同版本，你那代码很可能就失效了。就不要这么干。改用 mutable。写的代码应当是能保证正常工作，而不是可能不会出问题。

## 为什么我用const int*指向一个int了，编译器还是允许我修改这个int？

_Why does the compiler allow me to change an int after I've pointed at it with a const int*?_

因为 `const int* p` 表示的是“p保证不改变*p”，而不是“*p不会发生改变”。 将一个 `const int*` 指向一个 int 并不会把这个 int 变成 const 的。不能通过这个 `const int*` 来修改这个 int，但是，如果别的地方有个普通的 int*，指向同一个 int，那就可以通过这个int*来修改了。例如：

```c++
void f(const int* p1, int* p2)
{
  int i = *p1;         // Get the (original) value of *p1
  *p2 = 7;             // If p1 == p2, this will also change *p1
  int j = *p1;         // Get the (possibly new) value of *p1
  if (i != j) {
    std::cout << "*p1 changed, but it didn't change via pointer p1!\n";
    assert(p1 == p2);  // This is the only way *p1 could be different
  }
}
int main()
{
  int x = 5;
  f(&x, &x);           // This is perfectly legal (and even moral!)
  // ...
}
```

注意，main() 和 f() 完全可能位于不同的编译单元里，甚至是在不同时间编译的。所以编译器无法检测出这种指针的问题，因此也不可能指定什么语言规则来禁止这种用法。实际上，他们也不想制定这种规则，因为通常来说它就是一种特性，允许你用不同的指针指向同一个东西。仅仅是某个指针保证它不会更改指向的内容，并不是保证这个内容本身无法被更改。

## "const Fred* p"表示*p不能变化吗？

_Does "const Fred* p" mean that *p can't change?_

不是！（和上面一个问题是类似的） `const Fred* p`表示通过 p 无法修改 Fred，但是可以通过其它不带 const 的方式获取这个对象（比如通过一个non-const指针Fred*）。举个例子，假如你有两个指针，一个 `const Fred* p` 一个 `Fred* q`，都指向同一个 Fred 对象，那么可以使用指针 q 来修改这个 Fred 对象，但是指针 p 就不行。

```c++
class Fred {
public:
  void inspect() const;   // A const member function
  void mutate();          // A non-const member function
};
int main()
{
  Fred f;
  const Fred* p = &f;
  Fred*       q = &f;
  p->inspect();    // Okay: No change to *p
  p->mutate();     // Error: Can't change *p via p
  q->inspect();    // Okay: q is allowed to inspect the object
  q->mutate();     // Okay: q is allowed to mutate the object
  f.inspect();     // Okay: f is allowed to inspect the object
  f.mutate();      // Okay: f is allowed to mutate the object
  // ...
}
```

## 为什么把一个Foo**转为const Foo**会报错？

___Why am I getting an error converting a Foo** → const Foo**?___

因为将 `Foo**` 转为 `const Foo**` 可能是非法并且危险的。 C++ 允许这种安全的类型转换：`Foo* → const Foo*`，但是如果你想把 `Foo**` 转成 `const Foo**` 就会报错。 这种错误是个好事，下面的例子解释了为什么。但先放一个通用的解决方案：把 `Foo**` 换成 `const Foo* const*`。

```c++
class Foo { /* ... */ };
void f(const Foo** p);
void g(const Foo* const* p);
int main()
{
  Foo** p = /*...*/;
  // ...
  f(p);  // ERROR: it's illegal and immoral to convert Foo** to const Foo**
  g(p);  // Okay: it's legal and moral to convert Foo** to const Foo* const*
  // ...
}
```

把 `Foo**` 转成 `const Foo**` 是危险的操作，因为它可能会让你不知不觉中修改了一个 const Foo 对象：

```c++
class Foo {
public:
  void modify();  // make some modification to the this object
};
int main()
{
  const Foo x;
  Foo* p;
  const Foo** q = &p;  // q now points to p; this is (fortunately!) an error
  *q = &x;             // p now points to x
  p->modify();         // Ouch: modifies a const Foo!!
  // ...
}
```

假如 `q = &p` 这行是合法的，q 就指向了 p。下一行，`*q = &x`，把p本身给改成了指向 x（因为`*q`就是`p`）。这就坏事了，因为把 const 限定给弄没了：p 是一个 Foo*，但是 x 是一个 const Foo。`p->modify()` 这行利用了 p 修改了它的指向，这就成了一个问题，因为最终是修改了一个 const Foo。 打个比方，如果你把一个罪犯伪装成合法的人，那么他就可以利用你对他的信任。这样不好。 感谢 C++ 不让你这么干：`q = &p`这行会被编译器报一个编译错误。谨记：请勿对这种编译错误使用指针转换。坚决不行。 （注意:还有一种情况和这个在原理上是类似的，就是禁止将 Derived** 转为 Base**。）