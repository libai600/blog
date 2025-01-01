---
title: 'CMake Policy 机制'
slug: cmake-policy
date: 2018-07-10
tags: [CMake]
---

CMake 这个工具本身在开发过程中，有时为了修复 bug 或改进功能，会调整一些命令的行为，可能会导致新版本和旧版本运行结果不一致，为了能够向后兼容，CMake 引入了一个叫做 policy 的概念。

## CMake Policy

当一次版本更新后，如果修改了某个已有命令的行为，导致和之前版本不一致，CMake 就会添加一条 policy。每条 policy 都会定义新旧两种模式，分别描述之前版本和此次更新版本后是如何处理的。每条 policy 都有唯一的 ID，相关的详细信息可以在 CMake 的官方文档中查询到。

例如 cmake3.1 发布后，添加了一条 ID 为 CMP0054 的 policy，相关描述如下：

- 旧模式：在 if 命令中，尝试将参数看作是变量名进行变量引用，无论是什么参数类型。
- 新模式：在 if 命令中，仅尝试将普通参数看作是变量名，引号参数和括号参数不被看作是变量名。


比如下面这个例子：

```
set(A B)
set(B hello)
if("B" STREQUAL "hello")
    message(YES)
endif()
```

在旧模式下，if 命令会尝试将参数 `"B"` 看作是变量名进行引用，又因为在这个例子中确实存在变量 B，所以 if 命令的判断结果为真。在新模式下，不再将引号参数 `"B"` 看作是变量名，而是按照字面值进行比较，所以判断结果为假。

当 CMakeLists.txt 中的命令调用涉及到某个 policy 的相关内容时，cmake 在运行时会报出警报信息。比如如果用 3.1 及之后版本的 cmake 运行上面例子中的内容，除了会输出 YES 外，还会报出类似下面的信息。

```
CMake Warning (dev) at CMakeLists.txt:21 (if):
  Policy CMP0054 is not set: Only interpret if() arguments as variables or
  keywords when unquoted.  Run "cmake --help-policy CMP0054" for policy
  details.  Use the cmake_policy command to set the policy and suppress this
  warning.

  Quoted variables like "B" will no longer be dereferenced when the policy is
  set to NEW.  Since the policy is not set the OLD behavior will be used.
This warning is for project developers.  Use -Wno-dev to suppress it.
```

可以看到，因为没有明确的指明按何种模式处理，cmake 为了保持兼容性，仍然按照旧模式进行了处理，并提示使用 cmake_policy 命令指明按何种方式处理。

## cmake_policy 命令

cmake_policy 命令用于指定使用 policy 的哪种模式。其中一种调用形式为，

```
cmake_policy(SET <id> <mode>)
```

用于显式的指明某个 policy 的处理模式。其中 `<id>` 为 policy 的 ID，`<mode>` 有两种，NEW 或者 OLD。比如，`cmake_policy(SET CMP0054 NEW)`，表示之后的代码都按照 CMP0054 的新模式进行处理，再遇到 CMP0054 相关的内容时，不再报出警告信息。

cmake_policy 还有一种调用形式，

```
cmake_policy(VERSION <version>)
```

例如 `cmake_policy(VERSION 3.1)`，表示当前项目配置信息都是针对 cmake3.1 版本，cmake 会将 3.1 及之前版本引入的 policy 都按照 NEW 模式处理，但 3.1 之后版本引入的 policy 仍然不明确指定。

`cmake_minimum_required()` 命令内部会自动调用 `cmake_policy(VERSION <version>)` 命令，所以如果指定了 `cmake_minimum_required(VERSION 3.1)`，不必再调用 `cmake_policy(VERSION 3.1)`，也无需再单独指定 CMP0054 的模式。

建议在项目顶层的 CMakeLists.txt 文件第一行就调用 `cmake_minimum_required` 指定版本兼容信息。除非临时构建比较旧的代码项目，否则，及时处理有关 policy 的警告信息，尽量编写无二义的 cmake 代码，尽量避免使用 `cmake_policy` 命令，尤其避免指定按 OLD 模式进行处理。
