---
title: 'C++11：override 和 final'
slug: cpp11-override-and-final
date: 2016-08-03
tags: [c++]
---

C++11 引入了 `override` 和 `final` 修饰符，用于对类继承和函数重载的一些限定。

## override

`override` 用于对于成员函数做限定，添加到函数声明的后面，表示这个成员函数是对某个基类虚函数的重载。例如：

```c++
struct Base
{
    virtual void foo();
};

struct Derived : public Base
{
    void foo() override;
};
```

如果某个成员函数被声明为 `override`，但不是对基类虚函数的重载，编译器就会报错。例如：

```c++
struct A
{
    virtual void foo();
    void bar();
};

struct B : public A
{
    virtual void foo() override;       // 正确，重载A::foo
    virtual void fooo() override;      // 错误，基类无此函数
    virtual void foo() const override; // 错误，与A::foo签名不符
    virtual void foo(int a) override;  // 错误，与A::foo签名不符
    virtual void bar() override;       // 错误，A::bar不是虚函数 
};
```

所以，`override` 能有效避免那些“本以为重载了，但其实没重载成功”的问题。就比如上面例子中的 `fooo()`，程序员可能本想重载 `A::foo()`，但不小心写错了函数名，如果没有添加 `override`，编译器不会报任何错误和警告，结果就是没重载成。在 `override` 引入 C++11 之前这种问题是比较难以发现的，所以也建议在所有重载基类虚函数的地方都添加 `override`。

## final

`final` 限定有两种用法，用于阻止虚函数被子类重载，或者阻止某个类被继承。

当用于成员函数时，必须是对虚函数使用，放在函数声明后面，表示该虚函数不允许被子类重载，一般用于继承结构中间的某一层上，如果有子类试图继续重载该函数，编译器就会报错。例如：

```c++
struct A
{
    virtual void foo();
};

struct B : public A
{
    void foo() final;
};

struct C : public B
{
    void foo(); // 错误，B::foo被声明为final
};
```

如果 B 的设计者希望 B 及所有 B 派生类的 `foo()` 接口行为一致，就可以在 B 中将 `foo` 声明为 `final`，当再有派生类比如 C 试图重载 `foo` 时，编译器就会报错。

当 `final` 用于类时，放在类名的后面，表示该类不允许被继承，如果有其它的类试图继承，编译器就会报错。例如：

```c++
struct A
{

};

struct B final : public A
{

};

struct C : public B
{
    // 错误，B被声明为final
};
```