---
title: '浅谈C++多线程编程：标准库中的多线程支持'
slug: cpp-multi-thread-coding
date: 2019-09-14
tags: [C++]
---

## 前言

### 进程和线程

**进程（Process）** 是程序执行时的实例，**线程（Thread）** 是进程中的一个执行流程。一个进程可以包含多个并发的线程，但至少包含一个线程。不同的线程可以并发的执行不同的任务。进程是操作系统进行资源分配的最小单位，线程是操作系统进行运算调度的最小单位。同一进程中的多条线程将共享该进程中的全部系统资源。

对于算法程序，在多核 CPU 上可以利用多线程编程技术，实行并行的计算，提高程序的运行速度。对于图形界面程序，可以利用多线程进行耗时逻辑的执行，防止图形界面失去响应。

### C++库支持

在 C++11 之前，C++ 标准库没有提供对多线程编程的支持，多线程的实现需要调用系统平台上提供的库，如 pthread，win32 库。从 C++11 开始，标准引入了对多线程的支持。除此之外，Boost 中的 thread 库也对多线程编程进行了跨平台的封装，适用性也很好，接口和功能大部分都与 C++ 标准库中的相同，但 Boost 提供的功能更丰富一些。

如果开发环境支持 C++11 之后标准的多线程特性，就可以直接使用标准库的中的接口，否则可以考虑使用 Boost，对于基本的多线程能编程，大部分情况下两者都可以互换，只是 namespace 不同。

## 使用线程
### thread类

创建和管理线程需要使用 `thread` 类，Boost 的实现是在 thread 库中，boost 的接口兼容 C++ 标准，并在标准之外扩展了一些功能。使用 `thread` 类需要包含的头文件如下：

标准库：`<thread>`  
Boost库：`<boost/thread/thread.hpp>`

`thread` 类实现了线程的表示，负责启动和管理线程对象，其接口定义大致简化如下：

```c++
class thread
{
public:
    thread(F f);                    // 传递可调用对象
    thread(F f, A1 a1, A2 a2,...);  // 传递可调用对象及参数
    bool joinable() const;          // 判断是否可以join
    void join();                    // 等待线程完成执行
    void detach();                  // 分离线程
    id get_id() const;              // 获取线程id

    static unsigned hardware_concurrency(); // 获取当前硬件支持的最大并发数
};
```

### 启动和等待

线程就是在另外的一个流程中运行一个函数，执行一段逻辑，因此需要传递给 `thread` 对象一个可以被调用的东西，如函数，仿函数，function 对象，lambda 表达式等等。被调用的对象通过 `thread` 类的构造函数传递，如果被调用的函数需要参数输入，可以直接按顺序在 `thread` 的构造函数中指定，也可以使用 lambda，bind 进行包装。

当 `thread` 对象被成功构造后，线程就开始执行了，但并不是立即执行，根据当前环境和操作系统的调度，执行的顺序可能会有差异，thread对象的构造顺序不代表线程开始执行的顺序。

`thread` 对象构造后，便可以使用成员函数 `join()` 等其结束，`join()` 函数会一直阻塞等待，直到线程执行完成。

示例：

```c++
#include <stdio.h>
#include <thread>
#include <functional>

using namespace std;

void th0() { printf("thread 0\n"); }
void th(int id) { printf("thread %d\n", id); }

int main()
{
    thread t0(th0);
    thread t1(th, 1);
    thread t2([](){ th(2); });
    thread t3(bind(th, 3));

    t0.join();
    t1.join();
    t2.join();
    t3.join();

    return 0;
}
```

注意，通过 `thread` 构造函数传递给被调用函数的参数都是按值拷贝的，如果想传递引用，需要使用 `std::ref`（或`boost::ref`）进行包装，const引用需要使用 `std::cref`。例如:

```c++
void foo(int &a)
{
    ......
    a = ...;
}

int main()
{
    int m = 0;
    thread t(foo, ref(m));
    t.join();
}
```

### 返回值

如果线程中的函数有返回值，要获取这个返回值需要使用 `future` 类和 `async` 函数。使用它们需要包含 boost 的`<boost/thread/future.hpp>` 或 STL 的 `<future>`。

`future` 是一个类模板，用于表示线程函数的返回值，`async()` 函数用于启动线程执行一个函数，并获取它的返回值`future` 对象。它们的接口定义简化如下：

```c++
enum class future_status { ready, timeout, deferred };

template<typename T>
class future
{
public:
    T get();      // 获得最终返回值
    void wait();  // 阻塞等待线程的执行
    future_status wait_for(chrono::duration time);   // 等待一段时间
    future_status wait_until(chrono::duration time); // 等待直到某个时间点
    bool valid(); // 检查返回值是否可用
    shared_future share(); // 获得对应的shared_future对象
};

enum class launch
{
    async = 1,
    deferred = 2,
    any = async | deferred
};

future async(Function&& fn, Arguments&&... arg);
future async(launch policy, Function&& fn, Arguments&&... arg);
```

`async` 函数有两种重载，第一种形式与 `thread` 类的构造函数相似，传入一个可执行对象和它的输入参数，并返回一个 `future` 对象代表这个函数的返回值。类模板 `future<T>` 的模板参数 `T` 代表返回值的类型。`future` 类的 `wait()` 函数会阻塞并等待对应线程执行完成，直到获得结果值。`get()` 函数用于获得最终的返回值，如果线程还未执行完成，它会自动调用 `wait()` 函数进行阻塞等待。

示例：

```c++
int foo(int n)
{
    int number = 0;
    for (int i = 0; i < n; ++i)
        number++;
    return number;
}

int main()
{
    // 创建线程执行foo函数并获取future对象
    auto f = async(foo, 100000); // f类型实际为future<int>
    f.wait(); // 阻塞等待直到foo执行结束，如果删除这句代码，之后的get函数会自动阻塞等待
    int res = f.get(); // 获得最终返回值
    printf("%d\n", res);
    return 0;
}
```

需要注意，每个 `future` 对象的 `get()` 函数只能被调用一次，再次调用的话其结果是不确定的，可能引发异常。函数 `valid()` 可以用于检测 `get()` 函数是否可用，当调用过一次 `get()` 后，`valid()` 就会始终返回 false。

`future` 还提供了两个函数 `wait_for` 和 `wait_until` 分别用于阻塞等待一段时间或等待直到某个时间点，返回值是 `future_status` 枚举类型，表示等待后结果的状态。

`async()` 的第二种重载形式可以指定一个 `launch` 枚举参数，表示启动线程的时机：

- `launch::async` 立即启动一个新的线程执行目标函数。
- `launch::deferred` 推迟目标函数的执行，直到调用future对象的get()或wait()是才启动线程执行。
- `launch::any` 由系统调度决定立即执行还是推迟执行，时机不确定。

例如:

```c++
auto f1 = async(launch::async, foo, 100000); // 立即启动新线程执行foo
auto f2 = async(launch::deferred, foo, 50000); // 推迟foo执行
f2.wait(); // 此时f2对应的线程才执行;
```

`async()` 的第一种形式，即不带 `launch` 参数的形式，其实就等同于使用了 `launch::any` 参数，启动时机由系统决定。实际开发时，如果算法依赖某些执行顺序，应当显式的指定 `launch::async` 或 `launch::deferred`。

`future` 类的 `share()` 函数可以返回一个与其对应的 `shared_future` 对象，`shared_future` 与 `future` 的接口和功能基本相同，不过 `shared_future` 的 `get()` 函数没有执行一次的限制，可以被多次调用。

### 分离

如果线程在运行结束之前，对应的 `thread` 对象被析构，析构函数会自动调用 `std::terminate()` 函数强行中止线程的运行，这是非常不安全的操作。

如果在启动一个线程后，不再关心这个线程的执行，也不需要等待其执行完成，可以调用 `detach()` 函数进行分离。分离后，意味着失去了对这个线程的控制，`thread` 对象也不再表示这个线程，此时即使该线程仍在运行，析构该 `thread` 对象也是安全的。并且分离后，`joinable` 函数始终返回 false，`join` 函数也不再起任何作用。

### 杂项

`thread` 类的静态函数 `hardware_concurrency`可 以用于获取当前环境硬件支持的最大并发数，通常也就是当前 CPU 支持的最大线程数量，与 CPU 物理核心数量有关。

除此之外，子命名空间 `this_thread` 内提供了几个常见功能函数，如下，

```c++
id get_id()   // 同thread成员函数，用于获取当前所在线程的id
yield()       // 使当前线程放弃系统调度的执行时间片，重新调度其它线程执行
sleep_for()   // 线程睡眠指定的一段时间
sleep_until() // 线程睡眠至指定的时间点
```

### 中断

`boost::thread` 提供了有关线程中断的机制，但标准库中没有，这部分是对 C++ 标准的扩展，所以这一节的内容仅适用于 Boost。

`boost::thread` 提供了两个有关中断的成员函数：

```c++
void interrupt() // 要求线程中断执行
bool interruption_requested() // 检查线程是否被要求中断
```

被中断的线程会抛出一个 `thread_interrupted` 异常，这是一个空类，并不是 `exception` 的子类。如果线程支持中断操作，应当在线程执行函数中捕获这个异常并进行处理，如果不处理，默认的操作是终止线程。

示例：

```c++
void to_interrupt()
{
    try
    {
        for (int i = 0; i < 10; ++i)
        {
            this_thread::sleep_for(chrono::milliseconds(200));// 睡眠200毫秒
            printf("i %d\n", i);
        }
    }
    catch(const thread_interrupted&) // 捕获中断异常
    {
        printf("interrupted\n");
    }
}

int main()
{
    thread t(to_interrupt);
    this_thread::sleep_for(chrono::seconds(1)); //主线程睡眠1秒
    t.interrupt(); // 请求中断
    assert(t.interruption_requested());
    t.join(); //因为线程已经中断，所以join会立刻返回
    return 0;
}
```

可能的输出结果如下：

```
i 0
i 1
i 2
i 3
interrupted
```

boost 中的线程并不是在任何时候都可以被中断，如果我们将 `to_interrupt` 函数中的 `sleep_for` 睡眠等待删掉，这个线程就不会被中断。

`boost::thread` 库预定义了若干个线程的中断点，只有当线程执行到中断点的时候才可以被中断，一个线程可以包含任意多个中断点。这些中断点都是函数，包括：

```c++
thread::join()
thread::try_join_for()
thread::try_join_until()
this_thread::sleep_for()
this_thread::sleep_until()
this_thread::interruption_point()
condition_variable::wait()
condition_variable::wait_for()
condition_variable::wait_until()
condition_variable_any::wait()
condition_variable_any::wait_for()
condition_variable_any::wait_until()
```

除 `this_thread::interruption_point()` 外，其它中断点都是某种形式的等待函数，表示当前线程在这些阻塞等待的情况下能够被中断。

`this_thread::interruption_point()` 与其它函数不同，它是一个特殊的中断点，这个函数本身不影响正常逻辑流程，不阻塞线程，只是起到了一个标签的作用，表示线程执行到这句代码的时候可以被中断。

默认情况下线程是允许被中断的，但也可以进行控制，在 `boost::this_thread` 中提供了一些接口用于控制线程是否可被中断。

- `interruption_enabled()` 函数可以检测线程在当前是否可被中断。
- `iterruption_requested()`函数检测当前线程时候被请求中断。
- `disable_interruption` 类是一个 RAII 类型的对象，它在构造时使线程不能被中断，析构时使线程可以被中断。在 `disable_interruption` 对象的生命期内，线程始终时不可中断的，除非使用了 `restore_interruption` 对象。
- `restore_interruption` 类只能和 `disable_interruption` 类配套使用，它在构造时使线程可以被中断，析构时使线程不能被中断。

## 数据竞争

当多个线程同时访问修改某个数据时，其结果可能是不确定的。例如以下代码：

```c++
static int number = 0;

void worker()
{
    for (int i = 0; i < 10000; ++i)
        number++;
}

int main()
{
    thread th0(worker);
    thread th1(worker);
    thread th2(worker);
    
    th0.join();
    th1.join();
    th2.join();
    
    printf("%d\n", number);
    return 0;
}
```

三个线程都对同一个静态变量 `number` 进行 10000 次的自增操作，但最终 `number` 的值极有可能并不是 30000，而且这段程序每次执行的结果也极有可能不一样。

这是因为C++代码最终被编译成 CPU 指令，但变量 `number` 自增操作并不能通过一条CPU指令就执行完成，尽管在代码中只是一行，实际CPU需要三条指令：

1. 将内存中number变量的值装载到CPU寄存器中；
2. 将寄存器内的值加一；
3. 将寄存器中的值写回到内存中变量number的地址。

每个线程执行 `number++` 时都会执行这三条指令，但因为是并发的执行，各个线程的123步骤就会有交叉，最终导致 `number` 的值不确定。在多核 CPU 上，情况会更加复杂，但结果也是不确定的。这种多个线程同时尝试修改一个数据的情况称为**数据竞争（Data Race）**。

### 线程安全

如果一个函数能够被多个线程同时并发的调用，并且能够正常执行，产生正确和稳定的结果，那么称这个函数是线程安全（Thread-Safe）的，否则就不是线程安全的。比如上例中的 `worker` 函数就不是线程安全的，因为它被多个线程并发执行时，因为数据竞争不能产生正确和稳定的结果。

对于类，如果多个线程能够并发的调用某个成员函数，即便使用的是同一个对象实例，也能正常运行，那么这个成员函数就是线程安全的。如果一个类的所有成员函数都是线程安全的，那么称这个类是线程安全的。例如，

```c++
class Foo
{
public:
    void f1() { m_number++; }
    void f2() { s_count++; }
    int max(int a, int b) { return a > b ? a : b; }
private:
    static int s_count;
    int m_number;
};
```

函数 `f1` 不是线程安全的，因为当多个线程同时调用同一个 `Foo` 对象实例的 `f1` 函数时，对成员变量 `m_count++` 的操作将导致数据竞争。函数 `f2` 也不是线程安全的，因为共享了静态变量 `s_count`。`max` 函数是线程安全的，因为它不依赖任何共享数据，不会产生数据竞争。

在多线程的程序设计中，如果某个函数可能被多个线程并发的调用，但又存在数据竞争，则需要通过其它手段保证其线程安全，常见的方法包括互斥锁和原子操作。

## 线程同步

线程同步就是指的协同线程的执行步骤，使并发执行的多个线程按顺序先后执行某些逻辑，以此保证线程安全，或者协调线程之前的执行逻辑。本节将介绍一些常见的线程同步方式。

### 互斥锁

互斥锁（Mutex）也称为互斥量，用于临界区的保护。不允许多个线程同时执行的代码段或访问的数据，称为临界区（Critical Section），之前例子中的 `number++` 就属于临界区代码。使用互斥锁可以保证同一时刻只有一个线程访问临界区。

`boost::mutex` 和 `std::mutex` 提供了互斥锁的功能。使用它们需要 `include <boost/thread/mutex.hpp>` 或 `<mutex>`。`mutex` 的成员函数有三个：

- `void lock()` 锁定互斥量
- `bool try_lock()` 尝试锁定互斥量
- `void unlock()` 解锁互斥量

`lock` 函数会使当前线程获得互斥量的所有权，即加锁，`unlock` 函数会使当前线程释放互斥量的所有权，即解锁。当调用 `lock` 函数时，如果这个互斥量已经被另一个线程加锁，则 `lock` 函数会一直阻塞当前线程，直到该互斥量被解锁。`try_lcok` 函数会尝试对互斥量加锁，如果该互斥量没有被其它线程锁定，则将其锁定并返回 `true`，如果该互斥量已经被另一个线程锁定，则直接返回 `false`，此时不会阻塞当前线程。

互斥锁自己本身是线程安全的。

使用互斥锁保护临界区代码时，首先应当定义一个能够被所有线程共享的互斥锁，在进入临界区代码之前调用 `lock` 函数进行锁定，退出临界区代码之后调用 `unlock` 函数进行解锁。例如，

```c++
static int number = 0;
static mutex m;

void worker()
{
    for (int i = 0; i < 10000; ++i)
    {
        m.lock(); // 加锁
        number++;
        m.unlock(); // 解锁
    }
}

int main(int argc, char *argv[])
{
    thread th0(worker);
    thread th1(worker);
    thread th2(worker);
    
    th0.join();
    th1.join();
    th2.join();
    
    printf("%d\n", number);
    return 0;
}
```

变量 `number` 被所有线程共享，所以对其操作的代码属于临界区代码，我们在 `number++` 前后分别加锁解锁，这样就保证了同一时刻只有一个线程能够执行这句代码。所以上例中的程序执行完成后 `number` 的值始终为 30000。

### 递归互斥锁

递归互斥锁（Recursive Mutex）是一类特殊的互斥锁，用于在函数嵌套的情况下对临界区的保护。

普通的 `mutex` 互斥锁对象有一个局限性，在同一个线程内，不能对已经被锁定的互斥锁对象再次加锁。比如，在某个时刻线程 A 已经锁定了某个互斥量对象，但还未释放，之后还是在这个线程 A 中，有一段代码又调用了 `lock` 函数对其进行加锁，这样的操作结果是不确定的，可能会引发死锁。

所以，如果函数存在嵌套或递归，则可能无法使用普通的互斥锁，假设存在这样一种情况：

```c++
mutex m;

void fn1()
{
    m.lock();
    ...... // 临界区
    m.unlock();
}

void fn2()
{
    m.lock();
    ......
    f1();
    ......
    m.unlock();
}
```

`fn1` 和 `fn2` 不允许被多个线程同时执行，`fn2` 又调用了 `fn1`，如果有线程 `thread(fn2)`，因为 `fn2` 对 `m` 加锁，释放之前又进入了 `fn1`，`fn1`又对 `m` 加锁，尽管 `lock()` 函数的调用不在同一函数中，但仍然是在同一线程中对同一 `mutex` 多次加锁，将导致不确定的结果。当代码逻辑较为复杂时，上述情况就有可能发生，解决办法就是用递归互斥锁。

递归锁功能和函数与普通的互斥锁相同，但递归锁允许同一个线程对其多次调用 `lock()` 函数加锁，不过相应的也需要多次调用 `unlock()` 进行解锁，并且 `unlock()` 的调用次数必须与 `lock()` 的调用次数相同，否则也会发生死锁。

### 读写锁

如果某个数据不允许多个线程同时进行修改，即写操作，但允许多个线程同时进行访问，即读操作，则可以使用读写锁进行同步保护。读写锁允许多个线程同时读取一个数据，但不允许同时修改一个数据，或者一边读一边写。

`boost::shared_mutex` 和 `std::shared_mutex` 实现了读写锁的功能。Boost 和 STL 将其称之为共享互斥锁，但实际大多数情况下用于对数据的读写保护。

共享互斥锁提供了两种锁定模式：一种是共享模式，即读模式；另一种是独占模式，即写模式。

同一时刻多个线程可以使用共享模式共同获得锁的所有权，但同一时刻最多只有一个线程可以使用独占模式获得锁的所有权，且这两种模式不能同时使用。如果一个线程用读模式加锁，其它线程仍然可以使用读模式再次加锁，但不能使用写模式加锁，直到所有读模式解锁；如果一个线程用写模式加锁，其它线程则不能以任何模式再次加锁，直到该写模式被释放。

`shared_mutex` 类的接口如下：

```c++
void lock() // 独占模式锁定
bool try_lock() // 尝试独占模式锁定
void unlock() // 独占模式解锁
void lock_shared() // 共享模式锁定
bool try_look_shared() // 尝试共享模式锁定
void unlock_shared() // 共享模式解锁
```

示例：

```c++
static int number = 0;
static shared_mutex rw_lock;

void set_num(int n)
{
    rw_lock.lock(); // 加写锁
    number = n;
    rw_lock.unlock(); // 解写锁
}

void print_num()
{
    rw_lock.lock_shared(); // 加读锁
    printf("number=%d\n", number);
    rw_lock.unlock_shared(); // 解读锁
}
```

### 包装器

所有的互斥锁的加锁和解锁动作都需要成对调用，如果加锁了但忘记解锁，就会导致互斥锁一直被占用，形成死锁。Boost 和 STL 提供了一些互斥锁的包装器类，可以方便的管理锁的状态，主要用于自动解锁。主要用到的类包括 `lock_guard`, `unique_lock` 和 `shared_lock`，这三个类都是模板包装类。

#### lock_guard

`lock_guard` 在构造时需要指定一个互斥锁对象，构造函数会调用这个互斥锁的 `lock()` 函数对其加锁，`lock_guard` 对象在析构时，会调用互斥锁的 `unlock()` 函数进行解锁。所以借助 `lock_guard` 可以保证加锁和解锁成对调用，避免忘记解锁。例如，

```c++
int g_i = 0;
mutex g_i_mutex;  // protects g_i

void safe_increment()
{
    lock_guard<mutex> locker(g_i_mutex);
    ++g_i;
    printf("%d\n", g_i);
}
```

当函数 `safe_increment()` 执行完成返回前，`locker` 对象会被自动析构，因此会自动对 `g_i_mutex` 解锁，保证了逻辑正确。

#### unique_lock

`unique_lock` 与 `lock_guard` 的功能基本相同，一般也可以使用 `unique_lock` 替换 `lock_guard`，但 `unique_lock` 的功能更强大，提供了一些复杂的功能，暂时不做讨论。

#### shared_lock

`shared_lock` 与 `lock_guard` 和 `unique_lock` 的功能类似，构造时自动加锁，析构时自动解锁，但 `shared_lock` 加锁是调用的是 `lock_shared()` 函数，解锁时调用的是 `unlock_shared()` 函数，因此用于共享模式的保护。例如，

```c++
static int number = 0;
static shared_mutex rw_mu;

void set_num(int n)
{
    unique_lock<shared_mutex> locker(rw_mu);
    number = n;
}

int get_num()
{
    shared_lock<shared_mutex> locker(rw_mu);
    return number;
}
```

上例中使用了 `unique_lock` 对写模式加锁，使用 `shared_lock` 对读模式加锁。在 `get_num()` 函数中，如果要单纯的使用 `shared_mutex`，则需要先加锁，对 `number` 变量进行复制，再解锁，再返回复制的值，多一次数据复制的成本，但是通过 `shared_lock` 可以很简单的实现自动解锁。

### 条件量和信号量

TBD

## 原子操作

原子操作指的是不可被分割的操作，原子操作不能被打断，它一旦开始，就一直执行到结束。在所有线程中，不可能观察到原子操作完成了一半的情况：它要么完成了，要么还未开始。所以，原子操作是线程安全的。

注：数据的原子操作实际涉及很多复杂的概念，包含不同的实现方法，并不是所有的原子操作都是绝对线程安全的。本节只讨论其中最为常用的情况，所涉及的内容是线程安全的。

Boost 和 STL 提供了 `atomic` 模板来实现数据的原子操作，可以把一些简单数据类型的操作原子化。

### atomic 模板类

`atomic` 模板类简化的定义如下：

```c++
template<typename T>
class atomic
{
public:
    atomic() = default;   // 默认构造
    explicit atomic(T v); // 初始值构造
    atomic(const atomic &) = delete;            // 禁止对象间复制
    atomic& operator=(const atomic&) = delete; // 禁止对象间赋值
    
    void store(T value); // 存值
    T operator=(T value); // 与store等价
    
    T load(); // 取值
    operator T() const; // 去load等价
    
    T exchange(T new_value); // 交换
    
    T& storage(); // 获取内部数据对象，非标准，仅boost
    T const& strage() const; // 同上
    
    bool is_lock_free(); // 是否为无锁实现
}
```

除此之外，`atomic` 模板针对整型和指针类型特化出一些函数，支持一些常见的操作。对于整形来说，就是对其进行算数或逻辑运算，对于指针来说，就是对其进行地址偏移，与普通的整形和指针行为一致。

```c++
T fetch_add(T val); // 算数加法，返回原值
T fetch_sub(T val); // 算数减法，返回原值
T fetch_and(T val); // 仅对整形特化，逻辑与，返回原值
T fetch_or(T val);  // 仅对整形特化，逻辑或，返回原值
T fetch_xor(T val); // 仅对整形特化，逻辑异或，返回原值
T operator++ ();     // 前置自增
T operator++ (int);  // 后置自增
T operator-- ();     // 前置自减
T operator-- (int);  // 后置自减
T operator+= (T val);
T operator-= (T val);
T operator&= (T val);
T operator|= (T val);
T operator^= (T val);
```

以上所有 atomic 的成员函数对数据的操作都是原子操作，是线程安全的。

注：实际上 atomic 的成员函数大都包含另外一个默认参数 `memory_order`，如果修改这个默认参数，则这些函数就不一定是线程安全的。本节讨论的都是使用默认memory_order的情况。

借助 `atomic` 模板类，可以把类型T的一些操作原子化，但不是什么类型的数据都可以，必须是 Trivially Copyable 的类型，也就是通常讲的 POD 平凡数据类型，包括：

- 普通标量类型（Scaler），如int, char, bool, 枚举，指针等；
- 仅包含标量类型成员的class/struct/union，并且使用默认的构造，拷贝，赋值，析构。

可以通过 `std::is_trivially_copyable` 来判断某个类型是否符合 Trivially Copyable，例如，

```c++
std::cout << std::is_trivially_copyable<int>::value << std::endl;
```

另外，`atomic` 还定义预定义了很多常用类型，以方便使用，比如：

```c++
typedef atomic< bool > atomic_bool;
typedef atomic< char > atomic_char;
typedef atomic< unsigned char > atomic_uchar;
typedef atomic< short > atomic_short;
typedef atomic< unsigned short > atomic_ushort;
typedef atomic< int > atomic_int;
typedef atomic< unsigned int > atomic_uint;
... // 等等诸如此类
```

### 基本使用

有两种形式创建 `atomic` 对象，默认构造和带初始值的构造。前者构造出来一个初值不确定的 `atomic` 对象，一般不推荐使用这种方式，应使用带初始值的构造，例如 `atomic_int a(0)`。

成员函数 `store()` 和 `load()` 用于对 `atomic` 内部值进行存取，`operator =` 和 `operator T` 分别与 `store()` 和 `load()` 的功能相同，只是为了简化使用。例如，

```c++
atomic_int a(0);
a.store(5);
printf("a + 1 = %d\n", a.load() + 1);
a = 10;
printf("a + 1 = %d\n", a + 1);
```

成员函数 `exchange` 用于交换两个值，它保存新值的同时会返回原值，例如，

```c++
a = 100;
int old = a.exchange(200); // 交换后a = 200, old = 100
```

`storage()` 函数可以获取 `atomic` 内部的真实数据引用，进而任意操作这个数据，但这些操作就不是原子操作了，在非并发环境中可以提高一定的执行效率，但在并发的环境中不能保证正确性。`storage()` 函数仅由 Boost 提供，不属于 C++ 标准。

`atomic` 为整形和指针类型进行了特化，增加了一些 `fetch_xxx` 函数用于算数或逻辑运算，但这些 `fetch` 函数的返回值都是原值而不是计算后的值。另外也重载了常见的运算操作符，如++, +=, -=等，使得整形 atomic 和普通的整形使用起来几乎相同。

### 非原子操作

但需要特别注意，通过 `atomic` 对象的成员函数对其进行的操作都是原子操作，是线程安全的，但这些操作组合起来并不是原子的。例如，如果原子类型`atomic<int> a` 是多个线程的共享数据，如果多个线程同时执行 `a = a + 5`，那么结果是不确定的，因为 `a = a + 5` 不是原子操作，这句代码相当于多个操作的组合，等同于如下代码：

```c++
int temp = a.load();
temp += 5;
a.store(temp);
```

即先取得 a 的值，使用临时变量保存，然后将这个临时变量加5，再将计算后的值赋给 a。因此 `a = a + 5` 不是原子操作，更不是线程安全的。只有 `atomic` 的成员函数才是原子操作，才是线程安全的。线程安全的写法为 `a += 5`，因为这句代码只调用了 `atomic` 对象的 `operator+=` 函数。

### Lock Free

简单来说，LockFree 就是指的不使用互斥锁。`atomic` 类的 `is_lock_free()` 函数用于检测当前 `atomic` 类型是否是无锁实现的，即没有借助任何互斥锁来实现其所有原子操作。通常来说 C++ 内置的普通标量类型对应的 `atomic` 都是无锁实现，如 `atomic_int`, `atomic_long`, `atomic_bool` 等等。

LockFree的 `atomic` 通过更底层的指令来实现原子操作，效率比使用互斥锁要高。因此，对于基础数据类型的线程安全，如果比较关心执行效率，应尽量使用 `atomic` 原子操作代替加锁机制。

## 其他注意事项

**1. 避免死锁**

死锁指的是多个线程相互竞争资源，导致始终阻塞的现象。例如，线程1锁住了数据A没有释放，又去请求数据B，而同一时间线程2锁住了数据B没有释放，又去请求数据A。这样两个线程始终阻塞在请求对方资源的步骤上，导致程序无法继续运行。在开发中应仔细梳理竞争条件，避免类似这种死锁现象的发生。

另外，加锁和解锁必须成对使用。如果一个线程锁住了某个数据，但退出时忘记了解锁，或者中断时没有解锁，也会导致后续逻辑始终无法访问这个数据，始终被阻塞，造成死锁。因此在复杂逻辑的函数中，比如有多个 return 出口的函数中，应尽量使用 lock_guard 等包装类，避免因没有解锁导致的死锁。

**2. 互斥锁本身必须是线程共享的**

错误写法：

```c++
void foo()
{
    mutex m;
    m.lock();
    ...... // 临界区
    m.unlock();
}
```

上例中的互斥锁 m 没有起到任何保护作用，因为变量 m 在函数内局部实例化的，每个线程执行 `foo` 时都各自创建了一个 m 对象。正确的写法应当将 m 的实例化放到 foo 外部。

**3. 尽量缩小临界区范围**

临界区内的代码不能被多个线程并行执行，因此临界区越大，执行效率越低。因此应当尽量减少临界区范围，不要对非临界区加锁。例如，

```c++
static int value = 0;
static shared_mutex m;
void foo(int i)
{
    m.lock()
    value += slow_func_1(slow_func_2(i % 10)); // 临界区
    m.unlock();
}
```

假设函数 `slow_func_1` 和 `slow_func_2` 是线程安全的，当多个线程同时执行函数 `foo` 时，因为一开始就将 m 锁定了，导致实际并没有任何并发计算产生，没有利用多线程提高计算效率。高效的写法应当把线程安全的 `slow_func_1` 和 `slow_func_2` 移到临界区外面：

```c++
void foo(int i)
{
    int temp = slow_func_1(slow_func_2(i % 10));
    m.lock();
    value += temp; // 临界区
    m.unlock();
}
```

**4. 使用 mutabe 关键字**

使用 `mutable` 关键字可以避免因操作互斥锁影响类成员函数的 const 属性。例如，

```c++
class Foo
{
public:
    int get() const 
    {
        shared_lock<shared_mutex> locker(m); 
        return value; 
    }
    void set(int i) 
    { 
        unique_lock<shared_mutex> locker(m);
        value = i; 
    }
private:
    mutable shared_mutex m;
    int value;
};
```

如果上例中的读写锁没有添加 `mutable` 关键字，那么 `get()` 函数将会编译报错，因为函数被定义为 `const` 的，但是却操作了读写锁。

**5. 局部静态变量的初始化**

多线程环境下局部静态变量的初始化需要特别注意，例如常见的单例模式的一种写法：

```c++
static Singleton* instance()
{
     static Singleton s;
     return &s;
}
```

又比如：

```c++
int value = 0;
void foo()
{
    static mutex locker;
    locker.lock();
    value++;
    locker.unlock();
}
```

在 C++11 之前，上述函数 `instance()` 和 `foo()` 都不是线程安全的，可能导致产生多个 `s` 和 `locker` 对象。C++11 对此专门做出了规定，以保证局部静态变量初始化的线程安全。但编译器的实现存在差异，GCC4.3 和 Visual Studio 2015 才支持这个特性，之前版本的编译器均不能保证线程安全。

**6. 有关 volatile**

关键字 `volatile` 仅仅能保证数据的可见性，保证数据只在内存中被读写，即保证及时将 CPU 内部 reg 或 cache 中的数据同步到内存，`volatile` 不能保证数据的原子操作，更不能保证数据访问线程安全。
