---
title: 'CMake：简单工程和基本概念'
slug: cmake-simple-project
date: 2018-03-22
tags: [CMake]
---

## 简单工程

首先建立一个工程目录 project_1，放入以下三个代码文件和 CMakeLists.txt。

**foo.h**

```c++
#ifndef FOO_H
#define FOO_H

void printLine(const char* msg);

#endif // FOO_H
```

**foo.cpp**

```c++
#include "foo.h"
#include <iostream>

void printLine(const char *msg)
{
    std::cout << msg << std::endl;
}
```

**main.cpp**

```c++
#include "foo.h"
int main(int argc, char *argv[])
{
    printLine("hello, cmake!");
    return 0;
}
```

**CMakeLists.txt**

```
cmake_minimum_required(VERSION 3.0)
project(P1 LANGUAGES CXX VERSION 0.0.1)

message(STATUS "Project: ${PROJECT_NAME}")
message(STATUS "Version: ${PROJECT_VERSION}")
message(STATUS "Source dir: ${PROJECT_SOURCE_DIR}")
message(STATUS "Binary dir: ${PROJECT_BINARY_DIR}")
message(STATUS "Generator: ${CMAKE_GENERATOR}")

add_executable(app foo.cpp foo.h main.cpp)
set_target_properties(app PROPERTIES OUTPUT_NAME hello)
```

开始编译这个工程。新建一个 build_1 目录，与工程目录 project_1 平级，用于工程的编译。然后进入到 build_1 目录中，运行以下命令：

```
cmake ../project_1
```

随后大概会看到如下输出：

```
$ cmake ../project_1
-- The CXX compiler identification is GNU 7.4.0
-- Check for working CXX compiler: /usr/bin/c++
-- Check for working CXX compiler: /usr/bin/c++ -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Project: P1
-- Version: 0.0.1
-- Source dir: /home/zldd/cmake/project_1
-- Binary dir: /home/zldd/cmake/build_1
-- Generator: Unix Makefiles
-- Configuring done
-- Generating done
-- Build files have been written to: /home/zldd/cmake/build_1
```

cmake 会为我们产生 makefile 文件，继续运行 `make` 进行编译。

```
$ make
Scanning dependencies of target app
[ 33%] Building CXX object CMakeFiles/app.dir/foo.cpp.o
[ 66%] Building CXX object CMakeFiles/app.dir/main.cpp.o
[100%] Linking CXX executable hello
[100%] Built target app
```

完成后在 build_1 目录中产生了一个名为 `hello` 的可执行文件。

和使用传统的 makefile 不同，cmake 配置的工程在编译构建时，能显示出大概的进度，并且信息简单易读。如果想显示具体的编译器调用命令，可以使用 `make VERBOSE=1`，这样可以显示更详细的过程信息，包括编译每个文件时的命令。

## 代码目录和构建目录

CMake 将工程目录，也就是 CMakeLists.txt 所在的目录称之为**代码目录**（source directory），将 cmake 运行时的工作目录称之为**构建目录**（binary directory）。

在编译代码工程时，如果代码目录与构建目录相同，称为**代码内构建**（in-source build），否则称为**代码外构建**（out-of-source build）。上面的例子中，使用的是代码外构建， project_1 是代码目录，build_1 是构建目录。

默认情况下，cmake 产生的 makefile文件，配置缓存文件，和编译器产生的 object 文件，以及最终的程序文件，都会生成到构建目录内。所以，代码内构建会导致代码目录非常不干净。如果项目代码使用 SVN/Git 等版本管理工具托管，那每次 commit 之前都需要把代码目录清理干净，显然非常不方便。使用代码外构建则能避免这一点，所有产生的文件都在构建目录内，代码目录能够始终保持干净。因此，在开发过程中，**尽量使用代码外构建。**

## CMakeLists.txt

CMakeLists.txt 是 cmake 工程的配置文件，其内容描述代码项目的构建规则。cmake 运行时，需要指定一个代码目录，cmake 会在这个路径下查找 CMakeLists.txt 文件，并根据其中的规则描述产生对应的 makefile。

### cmake_minimum_required 命令

cmake_minimum_required 命令用于指定 cmake 的最低版本。比如在上例的 CMakeLists.txt 中，第一行是：

```
cmake_minimum_required(VERSION 3.0)
```

表示这个工程需要 3.0 及以上版本的 cmake。因为 CMake 各版本间存在一些差异，CMake 强烈建议在工程 CMakeLists.txt 文件的最开始，通过这个命令指定一个最低兼容的版本。如果当前运行的 cmake 版本不符合要求则会直接报错并停止运行。

### project 命令

project 命令用于指定代码工程的基本信息，每个工程都需要调用此命令。project 命令的完整形式如下：

```
project(<PROJECT-NAME>
        [VERSION <major>[.<minor>[.<patch>[.<tweak>]]]]
        [DESCRIPTION <project-description-string>]
        [LANGUAGES <language-name>...])
```

第一个参数是项目名，随后可以跟三个可选的参数段。

`VERSION` 关键字后面的参数用来指定代码项目的版本，可以填写最多四段的版本号，必须都是数字，比如 2.11.3.1234。`DESCRIPTION` 关键字后面的参数用来提供一段项目的描述信息。项目的版本和描述信息其实对于编译并没有什么实际的作用和影响。

`LANGUAGES` 关键字后面的参数用来指定项目使用的语言，通常可以指定为 C 或 CXX，如果没有提供这个参数，默认为 C 和 CXX 都启用，通常情况下最好指定语言。cmake 会根据项目指定的语言，在系统上查找合适的编译器。在上面的例子中指定了语言为 CXX，因此 cmake 只会查找 C++ 编译器，并且在首次运行 cmake 时，会有类似下面的输出，表示查找到的编译器信息。

```
-- The CXX compiler identification is GNU 7.4.0
-- Check for working CXX compiler: /usr/bin/c++
-- Check for working CXX compiler: /usr/bin/c++ -- works
```

另外，project命令还会自动定义一些变量：


- `PROJECT_NAME` 存储当前工程的名称。
- `PROJECT_SOURCE_DIR` 存储当前工程的代码目录。
- `PROJECT_BINARY_DIR` 存储当前工程的构建目录。
- `PROJECT_VERSION` 存储当前工程的版本信息。


### message 命令

前面例子中，在调用 project 命令之后，又通过 message 命令打印了一些变量的值。与之前在 CMake 基本语法一节中介绍的略有不同，这次传递了一个 `STATUS` 参数。message 命令的完整形式为：

```
message([<mode>] "message to display" ...)
```

其中第一个 mode 参数是可选的，表示消息的种类，常见的几个 mode 如下：

- `STATUS`：表示一般的状态消息，输出时内容前面会添加两个横线，CMake 本身的大部分信息都通过这个形式输出。
- `WARNING`：表示输出的是警告消息，输出时会特别体现出来，并输出这条消息所在的文件和行号。
- `SEND_ERROR`：表示输出的是错误消息，与警告消息形式一样，只不过显示的是 Error 关键字。这类错误消息并不会使 cmake 停止运行，cmake 会继续处理剩余的内容，但最终不会产生 makefile 等输出文件。
- `FATAL_ERROR`：表示输出的是严重错误消息，cmake 会直接停止运行。

在 project 命令之后，调用了五次 message 命令，分别打印了工程名称，版本，代码目录，构建目录和 Generator。显示 Generator 时，我们引用了变量`CMAKE_GENERATOR`，这也是一个 CMake 预定义的变量，存储了当前使用的 Generator 名称。有关 Generator 的概念后面会解释。

## Target

CMake 将代码工程中需要编译生成的可执行程序，动态库，静态库等最终产出物统称为**Target**。

### add_executable 命令

add_executable 命令用于添加一个可执行程序类型的 target，常见调用形式如下：

```
add_executable(<name> [WIN32] [MACOSX_BUNDLE]
               source1 [source2 ...])
```

其中第一个参数用于指定 target 的名称，最后的参数用来指定编译这个 target 需要使用哪些文件。

`WIN32` 和 `MACOSX_BUNDLE` 是两个可选的参数。使用 `WIN32` 表示这个程序是 Windows 平台上的 GUI 程序，而不是普通的命令行程序。使用 `MACOSX_BUNDLE` 表示这个程序是 macOS 上的 bundle 程序。

前面例子中的：`add_executable(app foo.cpp foo.h main.cpp)` 表示增加一个可执行程序类型的 target，名为 app，需要用到的文件为 foo.cpp，foo.h 和 main.cpp。cmake 默认在当前处理的 CMakeList.txt 所在的目录内查找文件，因此可以直接指定文件名或者文件相对路径，当然也可以使用绝对路径。

target 名称是 target 的唯一标识，用于在整个代码工程中相互区分，同一个工程中不允许有重名的 target。

对于一般的 C/C++工程，定义 target 时可以只指定需要哪些源文件(.cpp .c)，头文件(.h)可以不指定，因为头文件不是编译器的直接输入，但可以一并提供，用于在一些 IDE 中显示。但是对于某些特殊类型的工程，比如基于 Qt 框架的工程，就需要在定义 target 时同时指定有哪些头文件，因为 Qt 需要对头文件进行预处理，这属于构建过程的一个步骤。

### Target 属性

target 具有很多属性，这些属性用于控制 target 的构建。

set_target_properties 命令用于设置 target 的属性，完成的形式如下：

```
set_target_properties(target1 target2 ...
                      PROPERTIES prop1 value1
                      prop2 value2 ...)
```

比如前面的例子中的：

```
set_target_properties(app PROPERTIES OUTPUT_NAME hello)
```

表示将 app 的 `OUTPUT_NAME` 属性设置成了 hello。`OUTPUT_NAME` 属性用于控制一个 target 最终文件的文件名，所以产生的可执行程序名为 hello。如果没有设置 `OUTPUT_NAME` 属性，它默认等于 target 的名字。

注意：cmake 会根据系统平台通用规则，自动调整 target 最终输出文件名，比如上例工程如果移植到 Windows 系统上，生成的可执行文件名为 hello.exe，但 Liunx 系统上可执行程序通常没有后缀名，生成的文件名为 hello。所以，在设置 `OUTPUT_NAME` 属性时，不需要提供文件名的前缀后缀。

## CMake Generator

cmake 根据 CMakeLists.txt 内的配置描述，来生成本地构建系统（Native Build System）的输入文件。本地构建系统指的是诸如 make，Visual Studio，XCode 等工具，而 makefile，VS 工程文件，XCode 工程文件则是它们的输入文件。为本地构建系统生成输入文件的功能称为 Generator，cmake 在处理代码工程时，根据当前指定的 Generator 来产生用户想要的文件。

cmake 支持的构建系统大概可以分为两类：命令行式的和集成开发环境的。命令行式的主要包括各类型的 make 工具，比如 Unix make，MSVC nmake，MinGW make 等。集成开发环境包括各版本的 Visual Studio 以及 XCode，Eclipse 等。因为一些本地构建系统是和系统平台相关的，所以 cmake 在各系统上提供的 Generator 可能不同。可以运行 `cmake --help` 来查看当前系统上支持的 Generator。

```
$ cmake --help
Usage
  ...
Options
  ...
Generators

The following generators are available on this platform:
  Unix Makefiles               = Generates standard UNIX makefiles.
  Ninja                        = Generates build.ninja files.
  Watcom WMake                 = Generates Watcom WMake makefiles.
  CodeBlocks - Ninja           = Generates CodeBlocks project files.
  CodeBlocks - Unix Makefiles  = Generates CodeBlocks project files.
  CodeLite - Ninja             = Generates CodeLite project files.
  CodeLite - Unix Makefiles    = Generates CodeLite project files.
  Sublime Text 2 - Ninja       = Generates Sublime Text 2 project files.
  Sublime Text 2 - Unix Makefiles
                               = Generates Sublime Text 2 project files.
  Kate - Ninja                 = Generates Kate project files.
  Kate - Unix Makefiles        = Generates Kate project files.
  Eclipse CDT4 - Ninja         = Generates Eclipse CDT 4.0 project files.
  Eclipse CDT4 - Unix Makefiles= Generates Eclipse CDT 4.0 project files.
  KDevelop3                    = Generates KDevelop 3 project files.
  KDevelop3 - Unix Makefiles   = Generates KDevelop 3 project files.
```

运行 cmake 时，可以使用 `-G` 参数指定 Generator，比如 `-G "Unix Makefiles"`。如果不指定，cmake 会采用默认的 Generator，linux/unix 系统上默认生成 Unix Makefiles，Windows 系统上默认生成 Visual Studio 工程文件。前面例子在运行 cmake 时，没有指定 Generator，所以默认生成了 makefile，等同于使用 `-G "Unix Makefiles"` 参数。

如果某个工程之前运行过 cmake，然后需要更换 Generator，那再次运行 cmake 前，需要清空构建目录内的文件，或者直接更换一个新的构建目录重新运行 cmake。

## 移植到 Windows 系统

我们把刚才的工程移植到 Windows上。

在 Windows 上，最为常用的是微软的 MSVC 环境，我们就采用这个环境构建。首先确保你的系统上安装了 cmake 和 Visual Studio，笔者采用的 cmake3.10.3 最高可支持 VS2017，系统上安装有 VS2010 和 VS2015。

MSVC 环境下我们有两种方式可用，一是通过 cmake 产生 VS 的工程文件，再用 VS 打开这个工程进行构建；二是让通过 cmake 产生 MSVC 的 makefile，再使用 nmake 工具进行构建。下面我们分别使用这两种方式构建我们的工程，全部采用代码外构建的方式。

### 使用 VS 工程

首先在 project_1 目录外创建 project_1_vs 作为构建目录，打开命令行工具，进入到 project_1_vs 内，运行：

```
cmake ..\project_1 -G "Visual Studio 14 2015"
```

这个命令通过参数 `-G "Visual Studio 14 2015"`，指定让 cmake 生成 VS2015 的工程文件。

如果你的系统上安装的是其他版本的 VS，可以通过 `cmake --help` 查看对应的 Generator 名称，并替换 `-G` 参数的内容，例如 VS2010 对应的是 `"Visual Studio 10 2010"`。如果 cmake 没有找到对应版本的 VS，会报出类似"Failed to run MSBuild command"的错误。

运行成功后，可以看到构建目录内产生了一个 VS 的 solution 文件，之后就可以用 VS 打开直接编译了。如果不更改这个工程的输出选项，所有中间和结果文件同样也只会产生在构建目录中。

**注意：**无论 cmake 是否运行成功，都会通过文件缓存 Generator 的设置，如果需要更换 Generator，在运行 cmake 前，必须完全清空构建目录内的文件，或者直接更换一个新的构建目录重新运行 cmake。

### 使用 makefile 和 nmake

同样首先创建一个构建目录 project_1_make，然后通过 Visual Studio 命令提示符启动命令行环境。然后，进入到 project_1_make 目录中，运行：

```
cmake ..\project_1 -G "NMake Makefiles"
```

这次指定的 Generator 为 NMake Makefiles。完成之后 cmake 会产生相应的 makefile，然后就可以运行 nmake 进行构建了。