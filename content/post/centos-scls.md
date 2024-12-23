---
title: 'CentOS7/RHEL7 通过安装 SCLs 升级软件'
slug: centos-scls
date: 2022-05-05
tags: [Linux]
description: SCL 的安装和使用
---

在 RHEL 这种企业级的 Linux 发行版上，提供的基础软件工具和库往往只有很老的版本，CentOS 作为 RHEL 的复制品也是一样的情况。虽然 Red Hat 还在为这些被厂商抛弃的过时软件提供支持，修补安全漏洞等，但如果你的应用依赖新版软件和库，就得自己想办法了。

比如，作为 C++ 开发者，如果我们想使用一些比较新的 C++ 特性和标准库，就得自己动手额外安装一个新版本的 GCC。解决这个问题，常见的办法是下载一套 GCC 源代码，自己配环境编译和安装，比较费时费力，如果要想把 GDB 也一块升级那就更麻烦了。

还有一种更方便的办法，通过 [**Software Collections (SCLs)** ](https://www.softwarecollections.org/en/)，这是一个由 Red Had 支持的新软件包的源，同时也支持 CentOS, 为 CentOS 设立了仓库，安装和管理都和其它第三方仓库一样。

> **注意**   
> CentOS 7 已经于 2024 年 6 月 30 日正式 EOF 结束支持维护，其官方软件仓库已经下线，需要切换到其他镜像，如 [Aliyun 镜像](https://developer.aliyun.com/mirror/centos?spm=a2c6h.13651102.0.0.3e221b11Ln5gPX)。


# 使用 Software Collections

在 CentOS 上安装 Software Collections 的命令如下：

```
yum install centos-release-scl
```

在 RHEL 上则有些不同，具体请参考 https://github.com/sclorg/centos-release-scl
 
然后就可以像往常一样用 yum 安装新软件了，比如装个 Python3.5，

```
$ yum install rh-python35
```

最后，还需要一个命令使新版本的 Python 生效，

```
$ scl enable rh-python35 bash
```

运行以下命令可以验证一下是否生效了，

```
$ python --version
Python 3.5.1
$ which python
/opt/rh/rh-python35/root/usr/bin/python
```

通过 SCL 安装的软件路径在 /opt/rh 路径下，不会覆盖也不会影响系统上已经存在的版本 Python。`scl enable` 命令实际上是通过临时修改 PATH 环境变量让一个新工具生效的，所以在退出当前命令行后就会失效。

如果想每次进入命令行时能自动加载，可以在 ~/.bashrc 中 source 对应的 enable 脚本文件。比如把下面这行追加到 ~/.bashrc 文件中。

```
source /opt/rh/rh-python35/enable
```

这样每次都启动命令行都能加载上面安装的 Python3。

在命令行中执行 `scl enable rh-python35 bash` 和 `source /opt/rh/rh-python35/enable` 是相同的效果，但在 `.bashrc` 文件中只能使用后者。

# 通过 devtoolset 升级 GCC / GDB

SCL 提供了一个名为 **devtoolset** (Developer Toolset) 的系列工具包集合，包含了较新版本的 GCC，GDB 等 C/C++ 开发常用的工具，可以避免自己手动编译升级的麻烦。

devtoolset 本身分为多个大版本，在 CentOS 7 上有 devtoolset-7 到 devtoolset-11 多个版本提供，互不影响，可以共存。devtoolset 是开发工具的集合，以 devtoolset-9 为例，它包含了 GCC 9.3，GDB 8.3，还有 valgrind，binutils，elfutils，make 等常用工具。

## 安装 devtoolset

devtoolset 由 SCL 仓库提供，需要先按照前文的介绍安装配置好 SCL，然后就可以安装 devtoolset 了，比如：

```
$ sudo yum install devtoolset-9
```

以上命令安装的是 devtoolset-9，全部文件都会被安装在 /opt/rh/devtoolset-9 目录下，不会覆盖系统上现有的 GCC，GDB 等工具，这样保证了系统环境不受影响。安装后这些工具并没有生效，要启用 devtoolset-9，需要执行：

```
scl enable devtoolset-9 bash
```

再查看 GCC，就会发现已经变成了新的 GCC 9：

```
$ gcc --version
gcc (GCC) 9.3.1 20200408 (Red Hat 9.3.1-2)
$ which gcc
/opt/rh/devtoolset-9/root/usr/bin/gcc
```

如果想每次进入命令行时都能自动启用 devtoolset-9，在 ~/.bashrc 中增加以下内容：

```
$ source /opt/rh/devtoolset-9/enable
```

## 依赖与兼容性

我们使用 devtoolset-9 中的 GCC9 编译一个简单的程序。

```
$ source /opt/rh/devtoolset-9/enable
$ g++ main.cc -o a.out
```

使用 ldd 查看 a.out 的依赖：

```
$ ldd a.out 
    linux-vdso.so.1 =>  (0x00007fffa1388000)
    libstdc++.so.6 => /lib64/libstdc++.so.6 (0x00007f466d619000)
    libm.so.6 => /lib64/libm.so.6 (0x00007f466d317000)
    libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007f466d101000)
    libc.so.6 => /lib64/libc.so.6 (0x00007f466cd33000)
    /lib64/ld-linux-x86-64.so.2 (0x00007f466d921000)
```

会发现此时 a.out 链接的仍然是 /lib64 下面的 libstdc++ 和 libgcc_s，看起来好像有点不太正常，感觉使用新的 GCC 应该链接对应目录下的新的 libstdc++ 和 libgcc_s 才对，但其实不然，并且这是有意如此设计。

devtoolset-9 中的 GCC9 的确带了一个新的 libstdc++ 和 libgcc_s，位于 `/opt/rh/devtoolset-9/root/lib/gcc/x86_64-redhat-linux/9` 目录下，但仔细一看就会发现不对劲，这两个动态库居然只有一两百个字节大小，查看其内容就会发现，它们其实都是链接器脚本：

```
$ pwd
/opt/rh/devtoolset-9/root/lib/gcc/x86_64-redhat-linux/9

$ cat libstdc++.so
/* GNU ld script
   Use the shared library, but some functions are only in
   the static library, so try that secondarily.  */
OUTPUT_FORMAT(elf64-x86-64)
INPUT ( /usr/lib64/libstdc++.so.6 -lstdc++_nonshared )

$ cat libgcc_s.so
/* GNU ld script
   Use the shared library, but some functions are only in
   the static library, so try that secondarily.  */
OUTPUT_FORMAT(elf64-x86-64)
GROUP ( /lib64/libgcc_s.so.1 libgcc.a )
```

所以，这一点和自己由源码编译的 GCC 不同，源码编译的 GCC 通常会产生一个新的 libstdc++.so 动态库。但 devtoolset 中的 GCC 并没有真的附带一个 libstdc++.so 动态库，而是将那些相较于旧版 C++ 标准库（也就是系统中默认自带的 /lib64/libstdc++.so）更新多出来的内容，放在了附带的 libstdc++_nonshared.a 静态库中。也就是说，C++ 标准库中旧的原有的部分，依然链接 /lib64/libstdc++.so，而更新的部分，最终静态链接到程序中。libgcc_s.so 也是同理。

这样一来就最大程度保证了程序的兼容性，当 devtoolset 没有被启用时，编译出的 a.out 也能正常运行，并且在没有安装 devtoolset 的 CentOS 7 上，以及之后版本的 CentOS 系统上都能正常运行。

作为对比，如果使用手动源码编译的 GCC，要想实现同样的兼容性的话，通常需要在发布程序的时候附带一个新的 libstdc++.so，并且设置好程序的 RPATH 属性以保证优先使用这个库。


**参考资料**（*由于 CentOS 7 已经EOF，部分链接可能已失效。*）:

- https://github.com/sclorg
- https://www.linux.com/training-tutorials/best-third-party-repositories-centos/
- https://developers.redhat.com/products/developertoolset/overview
- https://www.softwarecollections.org
- https://pkgs.org/search/?q=devtoolset
