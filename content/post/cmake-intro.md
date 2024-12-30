---
title: 'CMake: 简介'
slug: cmake-intro
date: 2018-04-02
tags: [CMake]
---

## 什么是CMake

简而言之，CMake 是一个开源的，跨平台的自动化构建系统，用于软件项目的构建，测试和打包。CMake 最常用和最强大的功能是用来构建 C/C++ 软件项目，也就是编译源代码工程。

由源代码产生最终的可执行程序或程序库的过程就是软件项目的构建。Linux 平台下的 C/C++ 开发者应该都很熟悉 make 和 makefile。当一个源代码工程稍具规模后，手动调用 gcc g++ 等命令逐个编译的做法显然很不现实，这时我们通常会写一个 makefile，而 make 工具就是依照 makefile 中的规则，控制我们代码工程的编译。这样一来，省去了手动编译的麻烦，但编写 makefile 的任务自然还是需要开发者来完成，而 makefile 的问题就在于：项目一大就不好写。

当项目规模较大时，为了方便开发和维护，很可能我们的代码工程会分为许多子模块，子模块会被编译为库，而各个模块之间又存在依赖关系，同时也很可能还会加入许多用于测试的代码，这时候编写维护一套 makefile 就需要不少的精力。

如果这个项目又要做跨平台支持，需要同时发布 Windows，Linux 和 MacOS 的版本，就更麻烦了。各个平台的编译器都不同，甚至都不一定有好用的 make 工具，那最终很可能变成在 Linux 上用 makefile，在 Windows 上使用Visual Studio 工程，在 MacOS 上又使用 XCode 工程。每当项目有变化时，全都需要修改，相当的麻烦。

使用 CMake，可以解决以上难题。CMake支持平台无关的编译过程描述，也支持在某个平台上的特殊化处理。借助CMake，开发者可以只编写一套配置文件，然后在各种平台上构建我们的软件项目，无需再针对不同平台上的不同开发环境编写单独的配置格式。

实际上，CMake 并不直接调用编译器等工具，不直接控制源代码工程的编译构建过程，而是根据一个名为 CMakeLists.txt 的配置文件，生成各种本地构建系统的输入文件，CMakeLists.txt 中使用 CMake 的语法和命令描述构建过程，与具体的系统平台无关。

CMake 能够生成各种类型的 makefile 文件，比如 Linux 平台的 GNU makefile，Visual C++ 的NMake makefile，以及 MinGW 的 makefile。CMake 还能生成各种 IDE 的工程文件，比如 Visual Studio 或者 XCode 的工程文件，甚至连 Sublime Text 的 project 文件都可以。也就是说，CMake 为我们把各种构建环境封装了一层，因此无需再针对各种不同的构建环境编写 makefile 或创建 IDE 工程，我们需要做的只是编写一个 CMakeLists.txt，然后在不同的系统平台上使用 CMake 产生我们想要的 makefile 或工程文件，再使用对应的 make 工具或 IDE 来构建我们的项目。

*熟悉Qt的开发者可能会发现 CMake 和 qmake 很类似，没错，CMake 和 qmake 的基本原理是相同的，都是根据自己的一套配置文件生成各种本地构建环境的配置，但 CMake 的功能比 qmake 更强大。*

相比各种 makefile 和 autoconfig，CMake 的配置规则更简单，自动化程度更高，功能也更强大，所以即使我们的项目不需要做跨平台支持，使用 CMake 来管理源代码项目也要比 makefile 更方便高效。

**特点**

- 开源免费。CMake 使用 BSD 许可证发布，使用免费，且源代码开源。
- 使用简单。CMake 的配置规则书写要比 makefile 简单，自动化程度比 make 工具要高，可以替我们完成很多任务。
- 跨平台。CMake 本身支持 Windows/Linux/MacOS 等多种系统，可以生成多种类型的 makefile 和 IDE 的工程文件，适用于跨平台的软件项目。
- 可扩展。CMake 支持功能扩展，可以通过脚本扩展 CMake 的功能，实际上官方的发布包中就包含很多扩展的功能。

**适用场景**

- C/C++ 项目。目前，CMake 主要针对的是 C/C++ 代码项目，对于其他语言的项目并不适用。
- 中大型项目。CMake 的功能非常适用于管理中大规模的项目，可以非常方便的描述各个模块的依赖关系，组织不同模块的构建过程。
- 需要跨平台发布的项目。这点没有什么异议，这是 CMake 的绝对强项。


## 下载安装
[CMake 官网](https://cmake.org) 提供各版本的源代码和已经编译好的发布版，可以访问他们的下载页面（ https://cmake.org/download ），根据自己的操作系统下载。历史版本也可以在文件索引页面（ https://cmake.org/files/ ）里找到。

个人推荐下载对应系统的绿色版，解压即用。另外，最好不要使用比编译器还旧的 cmake 版本，否则一些特性可能无法支持，甚至出错。比如项目使用的是 VS2015，那就使用 VS2015 发布后更新的 cmake 版本。

下载后解压到合适的目录，再将 cmake 所在目录下的 bin 路径添加到环境变量 PATH 中。然后打开命令行工具，运行 `cmake --version`，检查是否能够找到 cmake 命令，如果提示找不到命令，请检查你的环境变量设置。

Linux 用户如果无法运行 cmake 命令，可能是系统版本较低，缺少相关依赖，可以通过 apt 或 yum 来安装 cmake。

CMake 提供了很详尽的使用手册和帮助文档，地址在 https://cmake.org/documentation/，另外，cmake 安装后也附带了离线网页文档，在安装路径下的 doc/cmake/html/index.html。

## 小试牛刀

我们的第一个工程在 Linux 上实现，开始前，确保 g++，make 和 cmake 已经安装准备妥当。这个工程很简单，源文件只有一个 main.cpp。

**main.cpp**

```c++
#include &lt;cstdio&gt;
int main(int argc, char *argv[])
{
    printf("hello, cmake!\n");
    return 0;
}
```

建立一个 simple_project 目录作为工程目录，将 main.cpp 放在这个目录下，然后再创建一个名为 CMakeLists.txt 的文本文件，内容如下：

**CMakeLists.txt**
```
cmake_minimum_required(VERSION 3.0)
project(P0 LANGUAGES CXX)
add_executable(app0 main.cpp)
```

现在，这个简单工程就准备好了，只有两个文件，main.cpp 是源文件，CMakeLists.txt 是 cmake的配置文件。下面开始编译这个工程。

创建一个 simple_build 目录，与工程目录 simple_project 平级，用于工程的编译。随后进入到 simple_build 目录内，运行以下命令：

```
cmake ../simple_project
```

随后大概会看到以下输出:

```
$ cmake ../simple_project
-- The CXX compiler identification is GNU 7.4.0
-- Check for working CXX compiler: /usr/bin/c++
-- Check for working CXX compiler: /usr/bin/c++ -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: /home/dan/cmake/simple_build
```

查看 simple_build 目录内的文件，就会发现 cmake 为我们生成了一个 Makefile。

```
$ ls
CMakeCache.txt  CMakeFiles  cmake_install.cmake  Makefile
```

继续运行 make 进行编译。

```
$ make
Scanning dependencies of target app0
[ 50%] Building CXX object CMakeFiles/app0.dir/main.cpp.o
[100%] Linking CXX executable app0
[100%] Built target app0
```

完成后再次查看目录内的文件，发现编译生成了可执行文件 app0。

```
$ ls
app0  CMakeCache.txt  CMakeFiles  cmake_install.cmake  Makefile
```

运行 app0，一切正常。

```
$ ./app0 
hello, cmake!
```

如果想重新编译，可以和往常一样，运行 `make clean` 后再重新运行 `make` 即可。