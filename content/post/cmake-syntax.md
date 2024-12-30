---
title: 'CMake：基础语法'
slug: cmake-syntax
date: 2018-04-15
tags: [CMake]
---

## 命令调用

cmake 的输入文件采用 CMake 的语法书写，由一条条命令调用组成。cmake 提供许多命令，开发者通过这些命令描述代码项目的构建过程，CMakeLists.txt 文件的内容就是对这些命令的调用。

命令调用格式为：

```
command_name(arg0 arg1 arg2 ...)
```

其中，command_name 是调用的命令名称，命令的名称是大小写无关的。括号内则是要传递给这个命令的参数，当存在多个参数时，使用空格隔开。如果参数较多，也可以使用换行符分隔，但左括号必须与命令名在同一行。

```
command_name(
    arg0 arg1 arg2
    arg3 arg4 arg5
    arg6 arg7 arg8
)
```

调用命令时，即使没有参数，小括号也不可省略。

## 注释

cmake 语法中，注释以 `#` 开头，一直到这一行的结尾，例如：

```
# 注释注释注释
add_executable(app main.cpp) # 注释注释注释
```

## message 信息输出

在后面的解释中，经常会用到 message 命令作为例子。message 命令可以接收任意个参数，将参数内容从左到右拼接后输出到控制台上。例如：

```
message(hello)   # 输出 hello
message(A B C D) # 输出 ABCD
```

注意：输出的内容是直接拼接的，并没有添加空格。

## 参数类型

cmake 中，传递给命令的参数可以有三种类型：**普通参数**，**引号参数**，**括号参数**。cmake 的所有命令都支持这三种参数。

### 普通参数 Unquoted Argument

普通参数是最常见的参数类型，例如：

```
message(hello)
# hello 为普通参数
```

普通参数**不能**包含空格，双引号`"`，括号`()`，和井号`#`，除非使用反斜杠`\`进行转义，比如：

```
message(abc\ \"\(\)\#)
# 输出 abc "()#
```

### 引号参数 Quoted Argument

被一对双引号包含的参数称为引号参数。例如：

```
message("hello, world!")
# "hello, world!" 为引号参数
```

双引号内的内容是参数的值，双引号本身不属于参数值的一部分，不会被传递给调用的命令。引号参数内可以包含任意字符，包括空格和换行，但如果参数内容本身又含有双引号，需要使用反斜杠`\`进行转义。参数内也可以使用`\n \r \t`表示换行和制表符，或`\\`表示一个反斜杠本身。

```
message("First line.\nSecond line, and some special characters ( ) # : ; \\
Third line, and a double-quote \" .")
```

输出为：

```
First line.
Second line, and some special characters ( ) # : ; \
Third line, and a double-quote " .
```

### 括号参数 Bracket Argument

括号参数以两个连续的左方括号开始，以两个连续的右方括号结束，有些类似 Lua 语言中多行注释的语法。例如：

```
message([[hello]])
# [[hello]] 为括号参数
```

开始和结束的两个方括号之间也可以加入任意个等号，但前后的数量必须相同，例如：

```
message([==[hello]==])
```

括号参数内可以包含任意字符，包括空格和换行。开始和结束符不属于参数值的一部分，不会被传递给调用的命令。括号参数的值就是字面值，不会处理参数内的任何转义字符。另外，如果开始符后面紧跟一个换行，这个换行也会被忽略。例如：

```
message([[One Line.]])
message([=[
First Line. hello\n\tcmake
Second Line.]=])
```

输出为：

```
One Line.
First Line. hello\n\tcmake
Second Line.
```

如果参数内容本身包含右侧的结束符，需要调整两侧等号的数量以避开，例如：

```
message([==[abc]=]def]==])
# 输出 abc]=]def
```

注意：只有 3.0 以及更高版本的 cmake 支持括号参数，之前的版本会将其当作普通参数进行处理。

## 变量

CMake 本身提供了变量的概念，但与一般语言中变量的功能相差甚远，更类似于字符串替换宏。

### 定义变量

变量的定义和赋值需要使用 set 命令，set 命令有多种调用形式，最常用的一种是传递两个参数，第一个参数是变量名，第二个参数是变量的值。例如：

```
set(ABC hello)
```

表示定义一个名为 ABC 的变量，值为 hello。如果当前已经存在重名的变量，则会直接覆盖它的值。

CMake 中的变量名是大小写敏感的，ABC 和 abc 是完全不同的两个变量。理论上，变量名可以包含任意非空字符，但在实际使用时，强烈建议采用大部分语言的做法，即仅包含字母数字和下划线，不要起一些复杂蹩脚的名字。另外，CMake 预定义了许多变量，用于控制工程构建过程，大部分以`CMAKE_`，`CTEST_`和`PROJECT_`开头，还有一些表示系统平台的变量，比如`WIN32`，`MSVC`，`UNIX`等，需要注意一下自定义的变量名不要和这些预定义变量产生冲突。CMake 的文档中有列举所有预定义的变量，和它们的作用。

如果要删除一个变量的定义，可以使用 unset 命令并传入变量名。另外，使用 set 命令并且只传入变量名也可以删除变量。例如，以下两条命令都表示删除变量 ABC。

```
unset(ABC)
set(ABC)
```

### 引用变量

之所以使用变量，就是为了避免填写重复的参数，当某些相同的参数需要被传递给很多不同的命令时，就可以用一个变量替换这些重复的参数。CMake 语法中，使用 `${ }` 来引用一个变量的值，大括号中填入变量名。例如：

```
set(ABC hello)
message(${ABC}) # 引用变量 ABC 的值，输出结果为 hello
```

这种引用基本可以当作是类似于C语言中宏定义替换。变量的引用不必单独作为一个参数出现，而是可以出现在普通参数和引号参数内，也就是说，可以将变量的引用和其他内容拼接在一起。例如：

```
set(ABC 456)
message(123${ABC}789)
message("123 ${ABC} 789")
```

输出：

```
123456789
123 456 789
```

但如果是括号参数内包括 `${ABC}`，则不会被处理，因为括号参数始终保持字面值。在普通参数和引号参数内，如果不希望 `${ }`表示变量引用，可以进行转义处理。例如：

```
message([[123${ABC}789]])
message(123\${ABC}789)
# 以上两条命令输出都为 123${ABC}789
```

当某个变量未定义，或被定义为空值时，对其引用的结果为空，可能会引起一些奇怪的错误。例如：

```
set(A1 "")
set(A2 [[]])
message(${A0})   # error, A0未定义
message(${A1})   # error，A1为空
message(${A2})   # error，A2为空
message("${A0}") # ok，输出空行
```

上例中，变量 A0 未定义，所以 `${A0}` 为空，因此 `message(${A0})` 就等同于 `message()`，没有传递任何参数，但是 message 命令至少需要一个参数，所以最终导致运行错误。变量 A1 和 A2 被定义为空，同理也会导致错误。而 `message("${A0}")` 能够正常执行，因为引用后变成了 `message("")`，实际传递了一个空的引号参数，结果为打印输出一个空行。通过这个例子也可以看出来变量引用就是字符串的替换。

一般情况下，如果需要通过 message 命令观察某个变量的值，最好使用引号参数的形式，比如 `message("${A0}")`。这样一来即使变量 A0 没有定义或者为空，也不会引发错误，而是输出一个空行。

变量的引用也可以嵌套。例如：

```
set(INDEX 2)
set(DATA2 000)
message("${DATA${INDEX}}")
# 输出 000
```

嵌套引用时，由内到外依次引用。上例中首先引用变量 INDEX 的值，参数变为 `"${DATA2}"`，然后引用变量 DATA2 的值，最终参数为 `"000"`。

### 列表变量

cmake 提供存储列表的功能，可以将多个值保存在一个变量中，比如常用于保存多个源文件的文件名。定义一个列表同样需要使用set命令，但不同于一般变量的定义，定义列表时除第一个参数表示变量名外，后面可以添加任意个参数，每个参数都表示一个列表元素。例如：

```
set(SOURCES a.cpp b.cpp c.cpp) 
# 变量 SOURCE 包含三个元素
add_executable(app ${SOURCES})
# 等同于 add_executable(app a.cpp b.cpp c.cpp)
```

实际上，cmake 中所有变量的类型都是字符串，并没有传统意义上的列表类型变量，而是通过以下两个特殊机制来实现类似“列表”变量的功能。

1. 定义变量时，如果提供了多个值，则使用分号作为分隔符，将它们拼接起来之后作为变量的值。
2. 传递参数时，如果一个普通参数中含有分号，则使用分号作为分隔符，将这个参数拆分成多个普通参数。

上例中，set 命令执行过后，变量 SOURCE 的值实际为 `a.cpp;b.cpp;c.cpp`。set命令使用分号将所有参数拼接起来作为 SOURCE 的值。add_executable 命令执行时，引用变量后变为 `add_executable(app a.cpp;b.cpp;c.cpp)`，然后按照分号进行拆分，最终等同于 `add_executable(app a.cpp b.cpp c.cpp)`。

定义列表变量时的拼接，是直接拼接的参数的值，不管参数本身中是否已经存在分号，也不管参数是什么类型。例如：

```
set(ABC a b c;d "e;f" [[g;h]])
# 变量 ABC 的值为 a;b;c;d;e;f;g;h
```

只有含有分号的普通参数才会被拆分，引号参数和括号参数即便含有分号也不会被拆分。如果不希望普通参数被拆分，可以使用反斜杠 `\` 对分号进行转义。例如：

```
message(a;b;c)     # 输出 abc
message(a\;b\;c)   # 输出 a;b;c
message("a;b;c")   # 输出 a;b;c
message([[a;b;c]]) # 输出 a;b;c
```

set 命令同其他命令一样，如果参数中含有变量引用，也需要先处理变量引用，利用这一点，可以追加列表元素。例如：

```
set(ABC a b)
set(ABC ${ABC} c d)
# 变量 ABC 的值为 a;b;c;d
```

### 环境变量

CMake 中也可以引用和设置环境变量。

环境变量的引用使用 `$ENV{VAR}`，例如:

```
#打印环境变量 PATH 的值
message($ENV{PATH})
```

设置环境变量同样使用 set 命令，并使用 `ENV{...}` 包含变量名，表示设置的是环境变量，例如:

```
set(ENV{PATH} $ENV{PATH}:/opt/bin)
```

## 条件判断

通过 cmake 配置代码项目时，可能需要一些条件判断来进行差异处理，比如在不同的系统平台上使用不同的参数设置。CMake 中使用 if / elseif / else / endif 命令进行条件判断，形式如下。

```
if(expression)
  #...
elseif(expression2)
  #...
else()
  #...
endif()
```

if 中的表达式有很多种形式，个人感觉设计的比较反人类，刚开始接触会很不适应。下面介绍一些比较常用的形式，其他形式可以自己查阅文档了解。

### 变量值判断

```
if(<argument>)
```

接收一个参数，根据参数值进行判断，判断采用如下规则：

首先进行常量判断（忽略大小写）：
1. 如果参数值为 1, ON, YES, TRUE, Y, 或者任意非零的数字，则判断结果为真。
2. 如果参数值为 0, OFF, NO, FALSE, N, IGNORE, NOTFOUND, 空字符串, 或者以 -NOTFOUND 结尾的值，则判断结果为假。

如果参数值不属于上面这些常量值中的任何一种，则该参数会被认为是变量名，进行变量判断（变量名大小写敏感）：

3. 如果该变量未定义，判断结果为假。
4. 如果该变量的值为 0, OFF, NO, FALSE, N, IGNORE, NOTFOUND, 空字符串, 或者以 -NOTFOUND 结尾的值，则判断结果为假，否则为真。

规则 1 和 2 容易理解，但如果和变量以及变量引用结合，就容易绕晕，看几个例子。

```
set(A NO)
set(B A)

if(A) # false
endif()

if(${A}) # false
endif()

if(B) # true
endif()

if(${B}) # false
endif()
```

`if(A)`，因为参数 A 不属于规则 1 和 2 中可以被直接判断的常量，所以 A 被当作变量名，又因为变量 A 的值为 NO，所以根据规则 4，最终判断结果为 false；

`if(${A})`，if 命令和其他命令一样，如果参数中含有变量引用，先处理变量引用。此例引用变量A后相当于`if(NO)`，根据规则2，参数 NO 属于能够直接判断的常量，结果为 false；

`if(B)`，和 `if(A)` 类似，参数 B 不属于规则 1 和 2 中可以被直接判断的常量，所以 B 被当作变量名，但因为变量 B 的值为 A，根据规则 4，不符合能够判为 false的条件，所以判断结果为 true；

`if(${B})`，因为变量 B 的值为 A，处理变量引用后相当于`if(A)`，参数 A 被当作变量名，最终结果和`if(A)`相同，为 false。

### 大小判断

```
if(<variable|string> <op> <variable|string>)
```

根据 `<op>` 的值不同，有两种判断：


1. 当 `<op>` 为 LESS, GREATER, EQUAL, LESS_EQUAL, GREATER_EQUAL 时，按数值判断大小关系。
2. 当 `<op>` 为 STRLESS, STRGREATER, STREQUAL, STRLESS_EQUAL, STRGREATER_EQUAL 时，按字符串判断大小关系。


执行时首先检查 op 前后的参数是否为已定义的变量名，如果是，则按变量的值进行处理，否则按参数字面值处理。例如：

```
set(A 1)
set(B 2)
if(A LESS B) # true
endif()
```

`if(A LESS B)` 中 A 和 B 都是已知的变量名，所以按它们的值进行判断，结果为真。但如果变量 A 和 B 都未定义，则判断结果为假，因为按参数字面值处理时，A 和 B 不是数字。

### 文件判断

```
if(EXISTS <path-to-file-or-directory>)
```

第一个参数为固定的 `EXISTS` 关键字，第二个参数可以是文件或目录的全路径，用于判断文件或目录是否存在，存在为真，否则为假。

### 与或非

if 命令中的表达式也可以使用 AND，OR，NOT 和括号进行组合，形成复杂的表达式。例如：

```
if(NOT ((1 LESS 2) AND (3 LESS 4)))
```

## 循环

### foreach 命令

foreach / endforeach 命令用于遍历一组值或基于范围的循环处理。

#### 遍历

用于遍历时的调用形式如下。

```
foreach(loop_var arg1 arg2 ...)
  # ...
endforeach()
```

foreach第一个参数为每次循环中用于表示当前遍历元素的变量名，其后的参数为要进行遍历的元素。例如以下命令执行后会依次输出 1 到 5。

```
foreach(item 1 2 3 4 5)
    message(${item})
endforeach()
```

结合变量引用，foreach 可以遍历列表变量的元素，例如：

```
set(A 1 2 3 4 5)
foreach(item ${A})
    message(${item})
endforeach()
```

#### 范围循环

范围循环有两种调用形式。

第一种：`foreach(loop_var RANGE total)`

表示从 0 循环到 total，例如下面命令会依次输出 1 到 10。

```
foreach(item RANGE 10)
    message(${item})
endforeach()
```

第二种：`foreach(loop_var RANGE start stop [step])`

表示从 start 循环到 stop，步进为 step，步进参数可选，默认为 1。例如下面命令输出 1 到 10 以内的奇数。

```
foreach(item RANGE 1 10 2)
    message(${item})
endforeach()
```

### while 命令

while / endwhile命令用于条件控制的循环，调用形式如下：

```
while(condition)
  # ...
endwhile()
```

其中 while 命令中的条件判断和 if 命令支持的相同。

### break/continue命令

break() 和continue() 命令与 C 语言中的类似，用于跳出循环和进行下一次循环，此处不再赘述。

## 数学运算

因为 cmake 的变量值都是字符串，所以并不能像其他语言那样直接对变量进行运算操作，而是需要用到 math 命令。math 命令的调用形式如下。

```
math(EXPR <output-variable> <math-expression>)
```

其中第一个参数 `EXPR` 关键字是固定的，第二个参数接收一个变量名，第三个参数是要运算的表达式，math 命令计算表达式后，将结果设置到第二个参数对应的变量中。例如

```
math(EXPR NUM "5*2")
# NUM值为10
```

目前，math 命令的功能还是比较简单，支持的运算符有 `+, -, *, /, %, |, &amp;, ^, ~, <<, >>, (, )`，它们的含义和C语言中的运算符相同。math 命令不支持大于小于等比较，不支持布尔表达式的运算，也不支持浮点类型的运算，只能够进行整形数值的运算。math 命令对表达式的检查也比较有限，某些情况下可能并不会报错。

另外，建议表达式都使用引号参数形式传递，因为当表达式中含有空格和括号时，如果使用普通参数传递需要进行转义处理，较为麻烦。

表达式中也可以包含变量引用，例如 `math(EXPR NUM "${NUM} + 1")`，实现对变量 NUM 的值加 1。