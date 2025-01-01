---
title: 'CMake：自定义构建步骤'
slug: cmake-custom-command-and-target
date: 2018-06-06
tags: [CMake]
---

本篇介绍如何通过自定义命令和自定义 Target 来实现某些特殊的构建步骤。

在实际的开发项目中，经常会有一些需要特殊处理的步骤，它们属于工程构建过程中的一环。比如，除了常规的编译所有 .c 文件外，可能需要使用脚本处理以及产生一些数据文件。CMake 可以自动处理常规的源文件编译的步骤，但那些额外的步骤就需要开发者自己来定义。CMake 提供了一些机制来支持这种自定义的构建步骤。

## 自定义命令

自定义命令有两种形式，一种用于产生文件，一种用于执行某些动作。我们先来看第一种。

### 自定义命令产生文件

这种形式可以理解为定义了一个构建步骤，目的是在构建过程中动态的产生一些文件给其他的 Target 构建使用，所以单独存在的 `add_custom_command` 没有实际作用，它是为其他 Target 服务的，需要搭配别的 Target 使用。

`add_custom_command` 的调用形式如下：

```
add_custom_command(
    OUTPUT output1 [output2 ...]
    COMMAND command1 [ARGS] [args1...]
    [COMMAND command2 [ARGS] [args2...] ...]
    [WORKING_DIRECTORY dir]
    [DEPENDS [depends...]] 
    [COMMENT comment]
)
```

完整的命令形式比较复杂，这里我们只列出了常用的部分，更详细完整的说明可以参考 CMake 手册。

`OUTPUT` 参数指定的是期望的输出文件，可以指定多个。这里期望的输出文件指的是后面给其他 Target 使用的文件，如果其他 Target 的文件列表中列出了这些文件，构建时 cmake 会先执行这个自定义命令来产生这些文件。如果命令本身还产生了一些临时文件或无用的文件，不需要在此列出。

`COMMAND` 参数指定的是实际执行的命令和输入参数，可以是任何程序或脚本。可以通过多次指定 COMMAND 参数来指定一串的命令，构建时它们会依次执行。

`WORKING_DIRECTORY` 参数指定的是这些命令的运行目录，不指定时，默认为变量 `${CMAKE_CURRENT_BINARY_DIR}` 表示的路径，也就是当前 CMakeLists.txt 对应的构建目录。

`DEPENDS` 参数指定的是自定义命令的依赖项，可以是文件，也可以是其他 Target。如果依赖的文件发生变化，构建时会重新执行这条自定义命令。如果依赖了其他的 Target，cmake 会保证构建的顺序，并且当这个 target 发生变化后，也会重新执行这条自定义命令。

`COMMENT` 参数用来提供一条消息，这条消息会在执行这个自定义命令的时候显示出来。

来看一个例子

**gen_foo.py**

```c++
str = """
#include <stdio.h>
void foo() {
    printf("%s\\n", "foooooooooooo");
}
"""

with open('foo.cpp', 'w') as f:
    f.write(str)
```

**main.cpp**

```c++
extern void foo();
int main(int argc, char *argv[])
{
    foo();
    return 0;
}
```

**CMakeLists.txt**

```
cmake_minimum_required(VERSION 3.5)
project(test LANGUAGES CXX C)

add_custom_command(
    OUTPUT foo.cpp
    COMMAND python ${CMAKE_CURRENT_SOURCE_DIR}/gen_foo.py
    DEPENDS gen_foo.py
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "generating foo.cpp"
)

add_executable(app main.cpp foo.cpp)
```

代码工程是构建一个名为 app 的可执行程序，源代码文件为 main.cpp 和 foo.cpp，其中 foo.cpp 是需要用自定义命令生成的，Python 脚本 gen_foo.py 就是用来产生 foo.cpp 的。当然实际的项目中不可能这样费尽周折产生一个固定的文件，一般都是读入某些文件，根据内容处理后再产生某些文件，我们使用这个简单的脚本仅用作举例解释自定义命令的用法。

在自定义命令中，我们指定期望的输出为 foo.cpp，因为在后面的 `add_executable(app main.cpp foo.cpp)` 中使用了 foo.cpp，所以在构建 app 之前会先运行自定义命令。假如没有任何 Target 使用 foo.cpp，这条自定义命令不会被执行，这就是前面说到的自定义命令需要搭配其他 Target 使用，单独存在没有意义。

我们需要通过脚本 gen_foo.py 产生 foo.cpp，所以 `COMMAND` 参数也就是对这个脚本的调用，并且通过 `WORKING_DIRECTORY` 指定了在构建目录中运行。另外，通过 `DEPENDS` 参数指定了依赖于 gen_foo.py 脚本文件，所以当这个脚本文件被更新后，构建时会重新执行这条自定义命令。

需要注意的是，使用自定义命令的 Target 和这个自定义的命令必须定义在同一个 CMakeLists.txt 文件中，比如上面例子中的 `add_executable` 和 `add_custom_command` 必须在同一个 CMakeLists.txt 中，否则 cmake 将无法识别处理。

### 执行额外动作

这是 `add_custom_command` 的另一种形式，目的是在某个 Target 构建之前或之后执行一些动作。这种形式的参数如下：

```
add_custom_command(
    TARGET <target>
    PRE_BUILD | PRE_LINK | POST_BUILD
    COMMAND command1 [ARGS] [args1...]
    [COMMAND command2 [ARGS] [args2...] ...]
    [WORKING_DIRECTORY dir]
    [COMMENT comment]
)
```

`TARGET` 指定这个命令是用于哪个 Target。这个 Target 必须和这条自定义命令定义在同一个 CMakeLists.txt 中。

`PRE_BUILD`，`PRE_LINK`，`POST_BUILD` 这三个参数用于指定命令何时执行，`POST_BUILD` 表示在对应的 Target 构建完成后执行，`PRE_LINK` 表示在 Target 完成编译之后但链接之前执行。`PRE_BUILD` 比较特殊，表示在 Target 构建之前执行，但在写这篇文章时，它只支持 Generate 为 Visual Studio的情况，其他情况和 PRE_LINK 相同，猜测以后可能会更新。

`COMMAND`，`WORKING_DIRECTORY`，`COMMENT` 这三个参数和 `add_custom_command` 的第一种调用形式中的参数作用相同。

这种形式下，当重新运行 make 时，如果对应的 Target 没有变化没有被重新构建，这条命令也不会被执行。

来看一个例子：

```
add_executable(app main.cpp)
add_custom_command(
    TARGET app
    POST_BUILD 
    COMMAND ${CMAKE_CURRENT_BINARY_DIR}/app
    COMMENT "run app...")
```

这里先定义了一个简单的可执行程序 app，然后增加了一个自定义命令，指定其在 app 构建之后运行，运行的命令就是 app 这个程序本身。通过这个设置，使得每次 app 在构建之后都运行一遍。

## 自定义Target

通过 `add_custom_target` 命令可以创建一个自定义的 Target。这个命令的参数比较多，我们这里只考虑比较常见的功能用法，以下是一个简化的定义。

```
add_custom_target(
    Name [ALL] [command1 [args1...]]
    [COMMAND command2 [args2...] ...]
    [DEPENDS depend depend depend ... ]
    [WORKING_DIRECTORY dir]
    [COMMENT comment]
)
```

第一个参数是自定义 Target 的名字，和其他所有 Target 一样，这个名字在整个工程中必须是唯一的。

`COMMAND`，`DEPENDS`，`WORKING_DIRECTORY`，`COMMENT` 和前面介绍的自定义命令中的参数含义基本一样。`COMMAND` 指定的是构建这个自定义 Target 需要执行的命令，可以指定多条，它们会依次执行。如果只有一条的话，也可以省略前面的 `COMMAND` 关键字。`WORKING_DIRECTORY` 指定这些命令的执行路径。`DEPENDS` 指定的是这个自定义 Target 的依赖项，可以是文件或者其他 Target，与自定义命令中的用法相同。

默认情况下，普通的构建流程不包含自定义的 Target，也就是说，如果运行完 cmake 再运行 make，并不会构建自定义的 Target。这种情况下，如果要构建自定义的 Target，需要在运行 make 时指定 target 的名字，比如自定义了一个名为 abc 的 Target，运行 make abc 就能构建它。除此之外，自定义 Target 时，指定 `ALL` 关键字参数可以使得默认构建流程包含这个 Target，这样在运行 make 的时候就能构建它了。

来看一个简单的例子：

```
add_custom_target(
    abc ALL
    COMMAND mkdir -p abc1
    COMMAND mkdir -p abc2
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "creating dir"
    )
```

我们自定义了一个名为 abc 的 Target，并指定了两条 mkdir 命令作为它的构建步骤。

再来看一个有依赖关系的例子：

```
add_executable(foo main.cpp)
add_custom_target(abc ALL
    COMMAND mkdir -p abc1
    COMMAND mkdir -p abc2
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "creating dir"
    DEPENDS foo
    )
```

这里的 `DEPENDS foo` 表示自定义 Target 依赖于另一个名为 foo 的 Target，cmake 会保证构建的顺序，当 abc 构建的时候，如果 foo 还没构建，则先构建 foo。
