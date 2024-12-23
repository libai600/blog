---
title: 'C++11：基于范围的 for 循环'
slug: cpp11-ranged-for-loop
date: 2016-07-15
tags: [C++]
---

C++11 引入了一种基于范围的 for 循环，简化和统一了数组和各种容器的遍历。

在 C++98/03 中，循环遍历数组和容器的写法不尽相同，并不统一。例如：

```c++
void print(int i) { std::cout << i << " "; }

int main()
{
    int arr[5] = { 1, 2, 3, 4, 5 };

    for (size_t i = 0; i < sizeof(arr) / sizeof(int); i++)
        print(arr[i]);
    std::cout << std::endl;

    for (int *i = arr; i < arr + sizeof(arr) / sizeof(arr[0]); i++)
        print(*i);
    std::cout << std::endl;

    std::vector<int> vec = { 1, 2, 3, 4, 5};

    for (auto i = vec.begin(); i < vec.end(); i++)
        print(*i);
    std::cout << std::endl;

    std::for_each(vec.begin(), vec.end(), print);
    std::cout << std::endl;
}
```

在这个例子中，我们使用下标和指针的方式遍历了一个数组，使用迭代器和 `for_each` 的方式遍历了一个容器。这些方式看起来都不够简洁，并且在遍历时都需要程序员明确指定循环的范围。但其实对于一个数组或容器的整体遍历来说，这个范围完全可以是“自说明”的，因为数组和容器本身就具有整体的范围信息，让程序员再提供其范围，既显得多余又容易出错。C++11 引入了基于范围的 for 循环，很好的解决了这个问题。我们使用新的写法重新来实现上面的例子。

```c++
int arr[5] = { 1, 2, 3, 4, 5 };
for(int i : arr)
    print(i);

std::vector<int> vec = { 1, 2, 3, 4, 5 };
for (int i : vec)
    print(i);
```

新的写法十分简洁统一，在 for 中，冒号前面表示每次循环的对象，冒号后面为循环的容器或数组。

如果需要对容器内的对象进行修改，可以使用引用来表示每次循环的对象：

```c++
std::vector<int> vec = { 1, 2, 3, 4, 5 };
for (int& i : vec)
    i += 10;
```

也可以使用常量引用来避免循环对象的复制：

```c++
std::vector<std::string> stringlist = { "abcdefg", "hijklmn", "opqrst" };
for (const std::string &str : stringlist)
    std::cout << str << std::endl;
```

还可以使用 `auto` 自动推导每次循环的对象类型：

```c++
std::vector<int> vec = { 1, 2, 3, 4, 5 };
for (auto i : vec)
    std::cout << i << std::endl;

std::vector<std::string> stringlist = { "abcdefg", "hijklmn", "opqrst" };
for (const auto &str : stringlist)
    std::cout << str << std::endl;
```

在循环中，也可以使用 `break` 跳出循环，以及使用 `continue` 跳到下一次循环。

这种基于范围的 for 循环也可以用来遍历像 `std::map` 这种关联性容器，但无论是什么容器，每次循环的对象都是容器的 `value_type`，对于 `map` 来说也就是 `std::pair`。例如：

```c++
std::map<int, std::string> dict = {{1, "a"}, {2, "b"}, {3, "c"}};
for (auto &item : dict)
{
    std::cout << item.first << item.second << std::endl;
}
```

基于范围的 for 循环可以总结为如下的形式：

```c++
for ( range_declaration : range_expression ) 
    loop_statement
```

它等价于下面的代码：

```c++
{
    auto && __range = range_expression;
    for (auto __begin = begin_expr, __end = end_expr; __begin != __end; ++__begin) {
        range_declaration = *__begin;
        loop_statement
    }
}
```

其中如果 `range_expression` 是一个数组类型，`begin_expr` 和 `end_expr` 就分别是数组的开始和结尾指针，也就是本文最开始例子中使用指针遍历数组的形式。如果 `range_expression` 是其他类型，`begin_expr` 和 `end_expr` 就分别是`__range.begin()` 和 `__range.end()`，也就是迭代器的形式。