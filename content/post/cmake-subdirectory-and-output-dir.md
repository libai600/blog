---
title: 'CMake：子目录工程和修改输出路径'
slug: cmake-subdirectory-and-output-dir
date: 2018-05-23
tags: [CMake]
---

本篇介绍如何通过 cmake 配置包含子目录工程的代码，以及如何修改 target 的生成路径。

## 子目录工程

当代码工程初具规模时，通常会将程序分为不同的模块，包括各种动态库，静态库，执行程序，测试程序等等，并且将各个模块的代码放到不同的目录中以便维护管理，形成一个按子目录组织的工程。

CMake 同样可以支持子目录工程的配置。CMake 是以 CMakeLists.txt 作为配置单元的，因此在每个子工程中，也需要一个 CMakeLists.txt 作为这个子工程的配置。添加子目录工程需要使用 `add_subdirectory` 命令，调用时指定一个路径名，cmake 会在这个路径下搜索 CMakeLists.txt 文件并进行处理。

来看例子，我们将之前创建和使用库的例子改为包含子目录的工程，工程结构组织如下：

```
代码目录
├── CMakeLists.txt
├── app
│   ├── CMakeLists.txt
│   └── main.cpp
└── student
    ├── CMakeLists.txt
    ├── student.cpp
    └── student.h
```

工程中包含两个模块，一个静态库，位于 student 目录下，一个执行程序，位于 app 目录下。代码目录中的 CMakeLists.txt 是顶层的配置文件，另外两个子目录中还各有一个 CMakeLists.txt，是子工程的配置文件。

**CMakeLists.txt**
```
cmake_minimum_required(VERSION 3.0)
project(sub_demo LANGUAGES CXX)
add_subdirectory(student)
add_subdirectory(app)
```

**student/CMakeLists.txt**
```
add_library(student STATIC student.cpp student.h)
target_include_directories(student PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
```

**app/CMakeLists.txt**
```
add_executable(app main.cpp)
target_link_libraries(app PRIVATE student)
```

在顶层的 CMakeLists.txt中，使用 `add_subdirectory` 命令增加了两个子工程，student 和 app，cmake 会在这两个路径下搜索 CMakeLists.txt 并读取其中的配置。这两个子工程一个是配置动态库，一个是配置执行程序。

在 student 的配置中，使用 `target_include_directories` 命令指定了 student 的包含路径。因为使用了 PUBLIC 参数，又因为 app 依赖 student 库，所以包含路径会传递给 app，app 在被编译时就能够找到 student 的头文件。变量 `CMAKE_CURRENT_SOURCE_DIR` 是 CMake 内置的变量，表示当前处理的 CMakeLists.txt 所在的路径。

创建构建目录并运行 cmake 和 make 编译，完成后查看一下构建目录内的文件。可以发现在构建目录中出现了和代码目录中同名的 student 和 app 目录，并且编译产生的动态库和执行程序分别位于这两个目录内，并没有像之前一样直接位于工程的构建目录内。

```
构建目录
├── app
│   └── app
└── student
    └── libstudent.a
```

### 子工程的代码目录和构建目录

在编译子目录工程时，cmake 会在顶层的构建目录内创建与代码目录一致的目录结构，各个子工程的编译输出文件会产生在对应的目录中。

我们之前了解到，CMake 预定义了变量 `PROJECT_SOURCE_DIR` 来表示工程的代码目录，和变量 `PROJECT_BINARY_DIR` 来表示工程的构建目录。同时，对于子工程的代码目录和构建目录，可以使用下面这两个变量获取，它们也是 CMake 内置的：

- `CMAKE_CURRENT_SOURCE_DIR` 表示当前 CMakeLists.txt 所在的目录
- `CMAKE_CURRENT_BINARY_DIR` 表示当前 CMakeLists.txt 对应的构建目录

例如，在前面例子中，如果在工程顶层的 CMakeLists.txt 内：  
`${CMAKE_CURRENT_SOURCE_DIR}` 等于 `${PROJECT_SOURCE_DIR}`  
`${CMAKE_CURRENT_BINARY_DIR}` 等于 `${PROJECT_BINARY_DIR}`

在 student 的 CMakeLists.txt 内：  
`${CMAKE_CURRENT_SOURCE_DIR}` 等于 `${PROJECT_SOURCE_DIR}/student`  
`${CMAKE_CURRENT_BINARY_DIR}` 等于 `${PROJECT_BINARY_DIR}/student`

## 自定义输出路径

默认情况下，在 CMakeLists.txt 中配置的target最终会产生在其对应的 `CMAKE_CURRENT_BINARY_DIR` 路径下。但在一些大型项目中，包含很多子项目，通常我们希望相同类型的模块输出到相同的目录中，比如所有动态库输出到一个目录，所有可执行程序输出到一个目录，所有测试程序输出到另一个目录等等。

CMake 中可以修改不同类型文件的默认输出路径，也可以为某个 target 单独修改输出目录。

### 按文件类型修改默认输出路径

CMake 提供了一些预定义的变量来控制不同类型文件的默认输出路径，常见的有：

- `CMAKE_ARCHIVE_OUTPUT_DIRECTORY` 表示静态库的默认输出路径
- `CMAKE_LIBRARY_OUTPUT_DIRECTORY` 表示linux上动态库（*.so）的输出路径
- `CMAKE_RUNTIME_OUTPUT_DIRECTORY` 表示可执行程序，以及windows上动态库（*.dll）的输出路径
- `CMAKE_PDB_OUTPUT_DIRECTORY` 表示MSVC环境下pdb文件的输出路径

同一个 target 有可能有不同类型的产出，比如在 MSVC 环境中，动态库 target 会产 .dll 文件，以及配套的 .lib 文件，两者就分别属于 RUNTIME 和 ARCHIVE 类型。另外需要注意，CMake 对动态库的处理有些特殊，虽然都是动态库，但在 linux 上属于 LIBRARY 类型，但在 windows 上则属于 RUNTIME 类型，需要用不同的变量进行指定。

修改以上这些变量可以影响所有 target 的输出路径，让它们不再产生到 `CMAKE_CURRENT_BINARY_DIR` 中。例如：

```
# set default output path
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/bin")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/lib")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/arch")
```

按照这样的配置后，所有的可执行程序默认产生到构建目录下的 bin 中，window 上动态库（\*.dll）也产生到 bin 内，linux 上动态库（\*.so）则产生到 lib，而所有的静态库产生到 arch 中。

### 修改target的输出路径

除了可以修改所有 target 的默认输出路径外，也可以单独指定某个 target 的输出路径，需要通过修改 target 的属性实现，常用的属性有：

- `ARCHIVE_OUTPUT_DIRECTORY`
- `LIBRARY_OUTPUT_DIRECTORY`
- `RUNTIME_OUTPUT_DIRECTORY`
- `PDB_OUTPUT_DIRECTORY`

它们的作用与前面那些用来控制默认输出路径的变量是一样的，只不过是对某个 target 起作用。实际上，CMake 是根据前面变量的值来初始化 target 对应的属性，比如根据变量 `CMAKE_LIBRARY_OUTPUT_DIRECTORY` 的值来初始化所有 target 的 `LIBRARY_OUTPUT_DIRECTORY` 属性值，但通过修改这些属性，有可以单独指定某个 target 的输出路径。

示例：

```
add_executable(abc main.cpp)
set_target_properties(abc PROPERTIES 
    RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/bin")

add_library(foo SHARED foo.cpp foo.h)
set_target_properties(foo PROPERTIES 
    RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/bin"
    LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/lib")
```