---
title: 'CMake：创建和使用库'
slug: cmake-create-and-link-library
date: 2018-04-12
tags: [CMake]
---

本篇介绍 cmake 中如何创建和使用库，包括动态库和静态库。

## add_library命令

创建库需要使用 add_library 命令，其常见形式如下：

```
add_library(<lib_name> <STATIC | SHARED | MODULE>
            source1 [source2 ...])
```

表示添加一个程序库 target，名为 lib_name，用到的文件为 source1 source2 等等。中间的参数用来指定这个库的类型：`STATIC` 表示静态库，`SHARED` 表示动态库，`MODULE` 表示插件库（特殊类型的动态库，一般用于运行时的显式加载调用）。

## target_link_libraries命令

链接库需要使用 target_link_libraries 命令，其常见形式如下：

```
target_link_libraries(<tt> [PRIVATE | PUBLIC | INTERFACE] <lib>...)
```

表示使 tt 链接 lib，tt 是某个 target 的名字，一般是执行程序或程序库类型的，lib 可以是库 target 的名字，也可以是某个现有的库文件的路径。

中间的关键字参数分别表示：

- `PUBLIC` 表示这条链接依赖关系不仅用于 tt 自身，也会传递到其他依赖 tt 的target。比如库 tt 依赖 libA，那么在产生 tt 时，会链接 libA，而项目中如果有其他模块又链接了 tt，那么也会一同链接 libA，即这种链接依赖关系是会传递的。
- `PRIVATE` 表示这条链接依赖关系仅用于 tt 自身，依赖关系不会传递。
- `INTERFACE` 表示这条链接依赖关系不用于 tt 自身的产生，反而仅用于其他需要依赖 tt 的target，INTERFACE 一般使用较少。中间的参数也可以省略，如果不指定的话默认按 PUBLIC 处理。

## 示例

项目中包含了两个 target，一个库，和一个调用库的执行程序。

**student.h**

```c++
#ifndef STUDENT_H
#define STUDENT_H

#include <string>

class Student
{
public:
    Student(const std::string& name, int age);
    void sayHi() const;

private:
    std::string m_name;
    int m_age;
};

#endif // STUDENT_H
```

**student.cpp**

```c++
#include <cstdio>
#include "student.h"

Student::Student(const std::string &name, int age)
    : m_name(name), m_age(age)
{
}

void Student::sayHi() const
{
    printf("Hi, I'm %s, %d years old.\n", m_name.c_str(), m_age);
}
```

**main.cpp**

```c++
#include "student.h"

int main(int argc, char *argv[])
{
    Student tom("Tom", 14);
    tom.sayHi();
    return 0;
}
```

**CMakeLists.txt**

```
cmake_minimum_required(VERSION 3.0)
project(P2 LANGUAGES CXX)

add_library(student STATIC student.cpp student.h)
add_executable(app main.cpp)
target_link_libraries(app PRIVATE student)
```

编译这个工程，创建一个构建目录，依次运行 cmake 和 make，可以发现在构建目录中产生了静态库文件 libstudent.a，和可执行程序 app。如果是在 windows 上采用 MSVC 环境，产生的静态库文件为 student.lib，测试程序为 app.exe。

```
$ cmake ../project_2
-- The CXX compiler identification is GNU 7.4.0
-- Check for working CXX compiler: /usr/bin/c++
-- Check for working CXX compiler: /usr/bin/c++ -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: /home/zldd/cmake/project_2_build
zldd@mate:~/cmake/project_2_build$ make
Scanning dependencies of target student
[ 25%] Building CXX object CMakeFiles/student.dir/student.cpp.o
[ 50%] Linking CXX static library libstudent.a
[ 50%] Built target student
Scanning dependencies of target app
[ 75%] Building CXX object CMakeFiles/app.dir/main.cpp.o
[100%] Linking CXX executable app
[100%] Built target app
zldd@mate:~/cmake/project_2_build$ ls
app  CMakeCache.txt  CMakeFiles  cmake_install.cmake  libstudent.a  Makefile
zldd@mate:~/cmake/project_2_build$ ./app 
Hi, I'm Tom, 14 years old.
```

来看一下这个 CMakeLists.txt 中的这三句配置。

```
add_library(student STATIC student.cpp student.h)
add_executable(app main.cpp)
target_link_libraries(app PRIVATE student)
```

首先，使用 `add_library` 命令，创建了一个程序库 target，名为 student，STATIC 关键字指定了它是一个静态库，所需的文件是 student.cpp 和 student.h。

然后，使用 `add_executable` 命令创建了一个可执行程序 app。这次的两个 target 我们都没有设置  `OUTPUT_NAME` 属性，产生的文件名默认和 target 的名字相同，但 cmake 会自动添加必要的文件名前缀或后缀。

最后，使用 `target_link_libraries` 命令，指定 app 需要链接 student。

cmake 在处理链接依赖时，能够根据项目中的配置，自动识别出 app 是一个可执行程序 target，student 是一个库 target，以及这个库的文件名，路径等，并且能自动产生链接时的具体参数。cmake 还能根据 target 之间的链接依赖关系，自动计算出所有 target 的编译顺序。甚至允许在指定链接依赖关系时某些 target 还未定义。比如，如果我们把之前的配置改成下面的写法，工程也能够正常编译链接。

```
add_executable(app main.cpp)
target_link_libraries(app student)
add_library(student STATIC student.cpp student.h)
```

如果希望 student 作为动态库，只需要将 STATIC 换成 SHARED 即可。（实际上可能还需要改动一些代码，比如 MSVC 环境中需要动态库导出符号）