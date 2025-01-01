---
title: 'CMake：构建类型和编译选项'
slug: cmake-build-type-and-compile-options
date: 2018-05-02
tags: [CMake]
---

本篇介绍 cmake 配置中如何指定工程的构建类型（debug / release）和添加自定义的编译选项。

## 构建类型

构建类型（Build Type）指的是编译构建一个代码工程时采用的配置。对于使用 IDE 的工程，一般可以在 IDE 内的选项上修改构建类型，例如 Visual Studio 工程中的 Configuration，默认分为 Debug 和 Release，在每次编译前可以选择使用哪一种。但对于命令行式的构建系统，一般需要自己调整 makefile 中的编译选项，来实现不同的构建类型。使用 cmake 可以不必手动修改编译选项，能够方便的切换构建类型。

### 设置构建类型

CMake 预先内置了四种构建类型：Debug，Release，RelWithDebInfo，MinSizeRel，可以满足大部分的使用情况，并通过预置的变量 `CMAKE_BUILD_TYPE` 表示当前的构建类型，可以通过修改它的值来改变构建类型，变量的初始值为空，表示不指定任何构建类型。

下面我们使用不同的构建模式来编译一个简单工程。

**CMakeLists.txt**

```
cmake_minimum_required(VERSION 3.0)
project(P1 LANGUAGES CXX)
set(CMAKE_BUILD_TYPE Debug)
message(STATUS "Build Type: ${CMAKE_BUILD_TYPE}")
add_executable(app main.cpp)
```

在这个工程中，我们使用 set 命令将变量 `CMAKE_BUILD_TYPE` 的值设置成 Debug，表示使用 debug 模式编译。需要注意，设置 `CMAKE_BUILD_TYPE` 要在添加 target 之前进行。

在构建目录中，运行 `cmake` 和 `make VERBOSE=1`，`VERBOSE=1` 可以使编译过程输出更详细的信息，包括编译器的调用参数。查看输出的信息，可以看到编译 main.cpp 时，添加了 `-g` 参数来产生调试信息，而不指定构建模式的时候是不会有 `-g` 参数的。

CMake 预置的构建模式对应的编译参数示例如下（Linux GCC环境）：

- 不指定：
c++ -o CMakeFiles/app.dir/main.cpp.o -c /home/zldd/cmake/project_1/main.cpp
- Debug：
c++ ***-g*** -o CMakeFiles/app.dir/main.cpp.o -c /home/zldd/cmake/project_1/main.cpp
- Release：
c++ ***-O3 -DNDEBUG*** -o CMakeFiles/app.dir/main.cpp.o -c /home/zldd/cmake/project_1/main.cpp
- RelWithDebInfo：
c++ ***-O2 -g -DNDEBUG*** -o CMakeFiles/app.dir/main.cpp.o -c /home/zldd/cmake/project_1/main.cpp
- MinSizeRel：
c++ ***-Os -DNDEBUG*** -o CMakeFiles/app.dir/main.cpp.o -c /home/zldd/cmake/project_1/main.cpp


除了在 CMakeLists.txt 中使用 set 命令设置 `CMAKE_BUILD_TYPE` 之外，还可以在运行 cmake 时直接指定。cmake 提供了一个 `-D` 参数，用来指定某个变量的初始值，调用格式为 `-D<variable_name>=<value>`。例如：

```
cmake ../source_dir -DCMAKE_BUILD_TYPE=Release
```

这样也可以设置构建类型。但请注意，`-D` 参数只是设置变量的初始值，如果在 CMakeLists.txt 中使用 set 命令再次修改了 `CMAKE_BUILD_TYPE`，那么构建类型以最后 set 修改的为准。

实际开发中，CMakeLists.txt 作为项目的配置，一般不宜经常修改，更常用 cmake 运行参数来控制构建类型。但我们可以在 CMakeLists.txt 中指定默认的构建类型，例如：

```
if(CMAKE_BUILD_TYPE STREQUAL "")
    set(CMAKE_BUILD_TYPE "Debug")
endif()
```

这样一来，如果没有用 `-D` 参数指明构建类型，则采用 Debug 构建。

## 系统平台相关性

有些时候，代码项目需要做跨平台支持，比如在 Windows 和 Linux 上都需要做编译。这时不可避免的需要做一些平台相关性的处理，CMake 提供了一些预置的变量来表示当前的系统平台或编译环境，当对应的环境匹配时被定义，且值为 True，常见的有：

- `WIN32`：表示当前系统为 Windows（包64位系统）。
- `UNIX`：表示当前系统为 Unix 以及 Unix-like 系统（包括 linux，cygwin 等）。
- `APPLE`：表示当前系统为 MacOS。
- `MSVC`：表示当前编译环境为 Microsoft Visual C++。
- `MINGW`：表示当前编译环境位 MinGW。

利用这些变量进行判断，就可以进行一些平台或环境的差异处理了。比如：

```
if(WIN32)
   ... ...
elseif(UNIX)
   ... ...
else()
```

## 自定义编译选项

除了可以让 cmake 根据构建类型自动调整编译选项外，也可以自己添加编译选项，可以对所有 target 添加全局的编译选项，也可以为某个 target 单独添加编译选项。

### 全局编译选项

添加全局编译选项使用 `add_compile_option` 命令，它接收任意个参数，在编译时，无论当前是什么构建类型，这些参数都会被传递给编译器。比如，GCC 默认不输出警告信息，可以通过这个命令添加开启警告的选项。

```
cmake_minimum_required(VERSION 3.0)
project(P1 LANGUAGES CXX)
set(CMAKE_BUILD_TYPE Debug)
message(STATUS "Build Type: ${CMAKE_BUILD_TYPE}")
if(UNIX)
    add_compile_options("-Wall" "-Wextra" "-Wpedantic")
endif()
add_executable(app main.cpp)
```

修改后的 CMakeLists.txt 中，通过 `add_compile_options` 命令添加了 `"-Wall" "-Wextra" "-Wpedantic"` 这三个编译选项，它们都是 gcc 的警告开关选项。再次运行 `cmake` 和 `make`，如果代码存在一些问题的话，输出中就会包含警告信息。

**需要注意：**`add_compile_options` 命令只对其后添加的 target 有效。也就是说，如果一个 target 是在调用 `add_compile_options` 之前添加的，那这些编译参数不会对这个 target 生效。

实际项目中，一般在配置的开始，在增加任何 target 之前使用 `add_compile_options` 命令，来设置一些全局的编译选项，比如为所有 target 打开编译警告输出。另外，除了可以添加全局的编译参数，还可以为某个target单独增加编译选项。

### Target编译选项

要为某个 target 添加编译选项需要使用 `target_compile_options` 命令，其完整形式如下：

```
target_compile_options(<target> [BEFORE]
  <INTERFACE|PUBLIC|PRIVATE> [items1...]
  [<INTERFACE|PUBLIC|PRIVATE> [items2...] ...])
```

例如，

```
target_compile_options(app PRIVATE "-Wall" "-Wextra" "-Wpedantic")
```

表示为 app 增加这三个警告选项。关键字 `PRIVATE` 表示这写选项仅对 app 自己生效（此命令中的 INTERFACE|PUBLIC|PRIVATE 参数与 target_link_libraries 命令中的意思一样）。可以一次性添加多个选项，也可以多次调用添加，比如，

```
target_compile_options(app PRIVATE "-Wall")
target_compile_options(app PRIVATE "-Wextra")
target_compile_options(app PRIVATE "-Wpedantic")
```

关键字 `BEFORE` 是可选的，表示添加的选项要放在所有编译选项的最前面，而不是追加。

一般来说，编译选项中，预定义宏（Predefined Macro）和包含路径（Include Path）这两个选项较为常见，CMake 提供了单独的命令来设置它们。

### 预定义宏（Predefined Macro）

同样，设置预定义宏也有两个命令，分别用于全局的和某个 target。

命令 `add_definitions` 用于添加全局的宏定义，对所有 target 都有效。例如：

```
# 预定义宏 AAA 和 BBB
add_definitions(-DAAA -DBBB)
# 带值的预定义宏
add_definitions(-DVERSION=123 -DNAME="string") 
```

命令 `target_compile_definitions` 用于给某个 target 添加预定义宏，形式与 `target_compile_options` 非常类似，例如：

```
# 预定义宏 AAA 和 BBB
target_compile_definitions(app PRIVATE AAA BBB) 
# 带值的预定义宏
target_compile_definitions(app PRIVATE VERSION=123 NAME="string")
```

其中，关键字 `PRIVATE` 表示仅对 app 自身有效。

**注意：**`add_definitions` 中的 `-D` 前缀不可省略，`target_compile_definitions` 中通常可以省略 `-D` 前缀。

### 包含路径（Include Path）

设置 include path 也有两个命令，分别用于全局的和某个 target。

命令 `include_directories` 用于添加全局的包含路径，对所有 target 都有效。例如：

```
# 添加路径 /opt/python3/include 到 include path
include_directories(/opt/python3/include) 
```

命令 `target_include_directories` 用于给某个 target 添加包含路径，例如：

```
# 为app添加包含路径 /opt/python3/include 到 include path
target_include_directories(app PRIVATE /opt/python3/include)
```

**注意：**如果路径中包含空格则需要使用双引号包含路径。

## 关键字 PRIVATE / PUBLIC / INTERFACE

在 CMake 中，一些针对 target 做设置的命令经常会用到 PRIVATE / PUBLIC / INTERFACE 关键字参数，在不同的命令中，它们表示的意思是一样的：

- `PUBLIC`：表示相关的设置不仅作用于当前指定的target，而且会随着依赖关系进行传递
- `PRIVATE`：表示相关的设置仅作用于当前指定的target，不会随依赖关系传递
- `INTERFACE`：表示相关的设置不作用于当前指定的target，但反而会随依赖关系传递

例如：

```
add_library(student SHARED student.cpp student.h)
add_executable(app main.cpp)
target_link_libraries(app student)
target_compile_definitions(student PRIVATE STU_MAKE_LIBRARY)
```

以上配置中，app 依赖 student，`target_compile_definitions` 命令为 student 添加了一个宏定义 `STU_MAKE_LIBRARY`，关键字 `PRIVATE` 表示这个宏定义仅对student有效。

如果将 `PRIVATE` 改为 `INTERFACE`，则正好相反，宏定义仅对依赖 student 的 target 有效，也就是 app，结果导致编译 student 库时编译选项中没有预定义 `STU_MAKE_LIBRARY`，而编译 app 时有。如果将 `PRIVATE` 改为 `PUBLIC`，则对 student 和 app 都有效。
