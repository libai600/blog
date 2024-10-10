---
title: 'C++11：统一的初始化和初始化列表'
slug: cpp11-uniform-initialization
date: 2016-07-10
tags: [c++]
---

在 C++98/03 中，有很多中不同的变量初始化方式。例如：

```c++
int i = 0;
int j(0);
int k = int(0);
std::string s1("abc");
std::string s2 = std::string("abc");
std::string s3 = "abc";
int arr[3] = {1, 2, 3};
```

这些各种各样的初始化方法，有的效果相同，有的效果相同但要求不同，基本都有各自的适用范围和作用，没有一种可以通用的方式，为了解决这个问题，C++ 提出了一种统一初始化（Uniform Initialization）的方式，它使用列表初始化来实现。

## 列表初始化

在 C++98/03 中，只有普通数组和 POD 类型的数据才可以使用初始化列表的方式。例如：

```c++
int arr1[3] = {1, 2, 3};
int arr2[] = {1, 2, 3, 4};

struct Data
{
    int a;
    int b;
} d = {1, 2};
```

除了可以继续用在这里提到的 POD 类型和数组，C++11 大大扩展了这种写法的适用范围。首先对于普通变量也可以使用大括号来初始化：

```c++
int i{0};
int j = {1};
```

对于类的构造也可以使用大括号：

```c++
std::string s1{"abc"};
std::string s2 = {"def"};
```

对于接收多个参数的构造函数，这种方式同样适用：

```c++
struct Foo
{
    Foo(int i, double d, char c) { }
};

Foo f1{1, 2.0, 'a'};
Foo f2 = {1, 2.0, 'a'};
```

在以上这些例子中，有没有等号都是一样的。

并且这种方式也可以用于 `new` 操作。例如：

```c++
int *i = new int{256};
Foo *f = new Foo{1, 2.0, 'c'};
```

甚至还可以直接为堆上分配的动态数组进行初始化：

```c++
int *arr = new int[3] {1, 2, 3};
```

初始化列表对于复杂类型也可以嵌套，比如对与前面例子中的 `struct Foo`，可以这么写：

```c++
Foo arr1[3] = {{1, 2.0, 'a'}, {2, 3.0, 'b'}, {3, 4.0, 'c'}};
Foo *arr2 = new Foo[3] {{1, 2.0, 'a'}, {2, 3.0, 'b'}, {3, 4.0, 'c'}};
```

对于 STL 的容器，同样可以使用列表初始化。例如：

```c++
std::vector<int> vec = {1, 2, 3, 4};
std::list<double> ls = {1.0, 2.0, 3.0};
std::map<int, char> dict = {{1, 'a'}, {2, 'c'}, {3, 'd'}};
```

总结来看，C++11 通过这种大括号的列表初始化的方式，形成了一套比较统一的初始化写法，也能够使很多代码变得更优雅易读。

## 防止类型收窄

使用列表初始化除了形式上比较统一以外，还有一个好处是可以防止类型收窄。类型收窄指的是在隐式的类型转换时，丢失了数据精度或者超出了表示范围。比如：

```c++
char c = 1024; // 超出表示范围
int i = 1.1;   // 丢失数据精度
```

这两行代码并不会导致编译报错（可能有编译警告），但显然执行的结果与程序员的本意不符。使用列表初始化的方式可以避免无意中的类型收窄：

```c++
char c {1024}; // Error
int i {1.1};   // Error
```

在使用列表初始化的时候，如果编译器检测到发生了类型收窄，就会报告编译错误。

## 自定义初始化列表

前面的例子我们看到，STL 的容器模板类可以使用不定长的初始化列表的方式来进行初始化，十分方便。

```c++
std::vector<int> vec = {1, 2, 3, 4};
std::list<double> ls = {1.0, 2.0, 3.0};
std::map<int, char> dict = {{1, 'a'}, {2, 'c'}, {3, 'd'}};
```

其实这种不定长的初始化列表是通过 `std::initializer_list` 来实现的，这是 C++11 新引入的一个模板类，通过将 `std::initializer_list` 作为构造函数的参数，我们自己的结构也可以实现通过不定长的初始化列表构造。

```c++
struct Foo
{
    Foo(std::initializer_list<int> il)
        : count(il.size()), sum(0)
    {
        for (auto i = il.begin(); i < il.end(); ++i)
            sum += *i;
    }

    int count;
    int sum;
};

int main()
{
    Foo f1 = {1, 2, 3}; // f1.count == 3, f1.sum == 6
    Foo f2 = {1, 3, 5, 7, 9}; // f2.count == 5, f2.sum == 25
}
```

在使用 `std::initializer_list` 时需要指定它的模板参数，也就是能够接受的参数的类型。在 Foo 构造时，大括号的列表参数会被转换为 `std::initializer_list` 传入构造函数，`std::initializer_list` 非常简单，它只有三个接口，`size()`用来获取参数列表的长度，`begin()` 和 `end()` 分别获取开始和结尾的迭代器。

实际上，`std::initializer_list` 不仅可以用作构造初始化，它可以作为任何函数的参数，所以我们也可以设计出能够接受不定长参数列表的函数接口。例如：

```c++
struct Foo
{
    Foo(std::initializer_list<int> il)
        : count(il.size()), sum(0)
    {
        for (auto i = il.begin(); i < il.end(); ++i)
            sum += *i;
    }

    void add(std::initializer_list<int> il)
    {
        count += il.size();
        for (auto i = il.begin(); i < il.end(); ++i)
            sum += *i;
    }

    int count;
    int sum;
};

int main()
{
    Foo f1 = {1, 2, 3}; // f1.count == 3, f1.sum == 6
    f1.add({4, 5}); // f1.count == 5, f1.sum == 15
}
```

## 成员变量就地初始化

C++11 标准引入了一条新的规则，允许非静态的成员变量在声明的时候就地初始化。

在 C++98 中，只有整形的静态成员常量才能在声明的时候使用等号=就地初始化，而普通的非静态成员变量只能在构造函数中进行初始化。例如：

```c++
struct Foo
{
    static const int a = 1; // OK: 整形的静态常量
    int b = 2; // Error in C++98
    Foo() : b(2) { } // b只能在构造函数中初始化 
};
```

在 C++11 中，对于所有的非静态成员变量，都允许就地初始化，可以使用普通等号赋值的形式，也可以使用初始化列表的形式。例如：

```c++
struct Foo
{
    int a = 0;
    int b {1};
    std::string name = "unknown";
    std::vector<int> v { 1, 2, 3, 4 };
}
```

对于有多个成员变量和多个构造函数的结构，通过成员就地初始化的方式可以简化其构造函数的编写。例如：

```c++
// C++98的写法
struct Foo
{
    // 每个构造函数都必须初始化所有成员变量
    Foo(int i) : data(i), ok(false), name("unknown") { }
    Foo(bool b) : data(0), ok(b), name("unknown") { }
    Foo(const char *cstr) : data(0), ok(false), name(cstr) { }

    int data;
    bool ok;
    std::string name;
};

// C++11成员就地初始化
struct Foo
{
    // 只需要在构造函数中初始化相应的成员，其余成员使用就地初始化
    Foo(int i) : data(i) { }
    Foo(bool b) : ok(b) { }
    Foo(const char *cstr) : name(cstr) { }

    int data = 0;
    bool ok = false;
    std::string name = "unknown";
};
```

需要注意，只有非静态成员变量才能就地初始化，静态的成员变量仍然需要在源文件中去定义。

```c++
struct Foo
{
    static int count = 0; // Error
    // 必须在cpp文件中定义：int Foo::count = 0;
}
```

另外需要注意的是，构造函数中的初始化可以“覆盖”成员变量的就地初始化，当一个成员变量同时存在就地初始化和构造函数的初始化时，最终效果为构造函数的效果。例如：

```c++
class Foo
{
public:
    Foo() : x(0), y(0), z(0) { }
private:
    int x = 1;
    int y = 1;
    int z = 1;
};
```

`Foo f` 的执行结果，成员 x, y, z 的值均为 0。