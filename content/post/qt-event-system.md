---
title: '浅谈Qt事件系统与事件编程'
slug: qt-event-system
date: 2018-04-02
tags: [c++, qt]
---

> Qt 的事件系统在 Qt 框架内发挥非常重要的作用，尤其是对与 Widget 图形框架来说更为重要，几乎所有的图形交互都是基于事件系统实现的，实际开发中，也经常需要围绕事件进行一些编程处理。

## 事件的表示

事件，顾名思义就是表示某件事情的发生。 Qt 使用 [QEvent](https://doc.qt.io/qt-5/qevent.html) 类来表示事件，一个事件对应一个 QEvent 对象。事件分为很多种类型，通过枚举 [enum QEvent::Type](https://doc.qt.io/qt-5/qevent.html#Type-enum) 来区分，可以通过函数 [QEvent::type()](https://doc.qt.io/qt-5/qevent.html#type) 来获取事件的类型。

大多数事件都有特定的派生类进行表示，比如 QMouseEvent, QKeyEvent, QPaintEvent 等等，它们都是 QEvent 的子类 。相较于 QEvent，这些子类增加了一些类型相关的信息，比如 QKeyEvent 的 `key()` 函数，用来获取按下了哪个键，或者 QMouseEvent 的 `pos()` 函数，用来获取鼠标的位置。

一些 QEvent 的子类可以用于表示多个类型的事件，比如，鼠标的按下，松开，移动，双击属于不同类型的事件，但都使用 QMouseEvent 这一个类表示，只不过它们的 `type()` 值是不同的。

事件可能在程序内部产生，也可能在程序外部产生，例如：

- QKeyEvent 和 QMouseEvent表示某种键盘和鼠标的操作，这些事件源自系统的窗口管理器；
- QTimerEvent 表示某个定时器触发了，这种事件通常来自操作系统；
- QChildEvent 表示某个 QObject 上的子对象被添加或删除了，这些事件来自程序自身内部。

当某个事件发生时，Qt 会根据事件的类型，创建一个对应的派生类对象来表示这个事件。程序外部产生的事件也会被转换为Qt的事件类型进行表示。函数 [QEvent::spontaneous()](https://doc.qt.io/qt-5/qevent.html#spontaneous) 的返回值表示这个事件是否源自程序外部。

## 事件的接收

在 Qt 中，事件的处理者是 [QObject](https://doc.qt.io/qt-5/qobject.html) 及其子类。Qt 中的所有事件最终都会被传递给某个 QObject 对象进行处理。QObject 及其所有子类都通过 [event()](https://doc.qt.io/qt-5/qobject.html#event) 函数来处理事件。

```c++
virtual bool event(QEvent* e)
```

对于 `QWidget` 而言，除了 `event()` 函数外，还定义了一系列的 `event handler` 函数，每个 `event handler` 专门处理一类事件，这部分我们在后面再详细介绍。

## 事件循环

事件产生后并不会立即被分发处理掉，而是被添加到一个队列中。事件系统中有一个循环逻辑，不停的在事件队列中取出事件，发送给对应的接收对象，这个循环就称之为 **事件循环（Event Loop）**。Qt 界面程序的基础就是这个事件循环，一个界面程序至少有一个事件循环，即主事件循环（Main Event Loop）。

事件循环可以近似简化为下面的逻辑：

```c++
while (!exit_was_called)
{
    while (!event_queue_is_empty)
        dispatch_next_event();

    wait_for_more_events();
}
```

`QApplication` 类的 [exec()](https://doc.qt.io/qt-5/qcoreapplication.html#exec) 函数的作用就是启动主事件循环，这个主事件循环会一直运行，直到调用了 [QCoreApplication::exit()](https://doc.qt.io/qt-5/qcoreapplication.html#exit) 使得事件循环退出。

在事件循环中，当队列中所有事件都处理完成后，`wait_for_more_events()` 会使得程序进入等待状态，直到有新的事件将程序唤醒。这个时候，程序内部的事件都已经处理完成了，将程序唤醒的都是外部事件。

我们来看一下比较典型的事件流程是什么样的。假设某个界面程序上有一个 QPushButton 按钮，它的 `clicked` 信号被连接到了一个 `doWork()` 槽函数上。当用户在按钮上点击鼠标时，操作系统会给程序发送一个鼠标点击的事件，事件循环被唤醒，并将其转换为 QMouseEvent 对象，然后添加到事件队列中。随后在下一次循环时，事件从队列中取出，发送给 QPushButton 对象进行处理，进而在 QPushButton 事件处理函数的内部触发了 `clicked` 信号，最终使得 `doWork()` 函数被调用。假设点击事件发生后没有其他外部事件产生，`doWork()`函数执行完成后，也没有产生新的内部事件，那么此时队列没有其他事件可以继续处理，事件循环再次进入等待状态，否则就继续处理队列中的其他事件，直到将事件队列为空。

在事件循环中，事件只能一个接一个的进行处理，在前面的事件处理完成之前，后续的事件只能在队列中等待。在上面的例子中，如果 `doWork()` 的执行时间很长会发生什么呢？事件循环必须等待 `doWork()` 函数执行完返回后才能处理下一个事件，在这期间内，事件循环相当于被卡住了。卡住的事件循环无法处理任何事件，也无法收到新的外部事件，所以此时在这个程序内，界面无法更新，计时器也无法触发，网络 IO 也得不到任何反馈，表面上看就是“卡住了”。并且系统的窗口管理器也能检测到程序不再处理任何事件了，所以提示用户“程序失去响应”。

因此，**在界面交互的编程中，不应在事件响应函数中执行任何耗时的逻辑，以免事件循环被阻塞使得程序失去响应**。

### 强制处理事件

在实际开发中，确实经常有一些很耗时的操作需求，比如复制大量文件，读写大量数据等等。这种情况一般有两种实现方案，一种是开启子线程进行耗时逻辑处理，避免主线程的事件循环阻塞，这种方式本文不做详细讨论，这里主要解释另一种：强制让事件循环处理事件。

调用 [QCoreApplication::processEvents()](https://doc.qt.io/qt-5/qcoreapplication.html#processEvents) 函数可以强制使事件循环分发处理队列中剩下的事件，直到队列中没有事件为止，或者直到给定的时间结束。

还是用前面的例子，假设 `doWork()` 函数是要向某个文件写入大量数据，执行时间很长，如果不采取特别的措施，在 `doWork()` 执行过程中肯定会阻塞事件循环，使界面失去响应。但如果在写入数据的过程中，没间隔固定一段，比如每写入 4MB，就调用一次 `processEvents()`，就相当于在这个非常耗时的逻辑中，不时的间隔暂定一下，转而去处理事件队列中的事件。这样一种方式，就能在执行耗时逻辑的同时，穿插处理事件，而不至于让界面失去响应。当然，这个间隔需要开发者自己把握，如果间隔太小调用太频繁，会导致效率降低，如果间隔太大调用又太少，还是会导致一段时间内无法无法处理事件，界面卡顿。

### 回到事件循环

在 Qt 的文档中，有时会遇到类似 *“回到事件循环（when control returns to the event loop）”* 的描述，它指的是当一个事件处理完成后，逻辑控制权重新回到了事件循环内的时刻，此时事件循环能够继续处理后续的事件。例如，在函数 [QObject::deleteLater()](https://doc.qt.io/qt-5/qobject.html#deleteLater) 的解释中就有这样的描述：*"The object will be deleted when control returns to the event loop."*

## 修改事件的处理逻辑

基于 Qt 开发时，经常会通过继承 Qt 的类来扩展出一些我们想要的功能，这其中一个重要手段就是修改对一些事件的处理。比如我们想让 QTreeView 对某些键盘操作有响应，一般只需要继承 QTreeView，并且修改对键盘事件的处理。

实际开发中，对某些事件的修改会比较常见，例如鼠标键盘的操作，窗口的显示和绘制，拖拽和释放动作，焦点的获取和丢失事件等等。对于这些事件的处理，QWidget 定义了一系列相应的虚函数，称之为 **Event Handler**，它们各自专门处理一类事件。

```c++
virtual void actionEvent(QActionEvent *event)
virtual void changeEvent(QEvent *event)
virtual void closeEvent(QCloseEvent *event)
virtual void contextMenuEvent(QContextMenuEvent *event)
virtual void dragEnterEvent(QDragEnterEvent *event)
virtual void dragLeaveEvent(QDragLeaveEvent *event)
virtual void dragMoveEvent(QDragMoveEvent *event)
virtual void dropEvent(QDropEvent *event)
virtual void enterEvent(QEvent *event)
virtual void focusInEvent(QFocusEvent *event)
virtual bool focusNextPrevChild(bool next)
virtual void focusOutEvent(QFocusEvent *event)
virtual void hideEvent(QHideEvent *event)
virtual void inputMethodEvent(QInputMethodEvent *event)
virtual void keyPressEvent(QKeyEvent *event)
virtual void keyReleaseEvent(QKeyEvent *event)
virtual void leaveEvent(QEvent *event)
virtual void mouseDoubleClickEvent(QMouseEvent *event)
virtual void mouseMoveEvent(QMouseEvent *event)
virtual void mousePressEvent(QMouseEvent *event)
virtual void mouseReleaseEvent(QMouseEvent *event)
virtual void moveEvent(QMoveEvent *event)
virtual void paintEvent(QPaintEvent *event)
virtual void resizeEvent(QResizeEvent *event)
virtual void showEvent(QShowEvent *event)
virtual void tabletEvent(QTabletEvent *event)
virtual void wheelEvent(QWheelEvent *event)
```

实际上，对于 QObject 和 QWidget 而言，前面提到的 `event()` 函数只是事件处理的最开始入口，它会根据事件的类型，再调用相应的 event handler 函数。`QWidget::event()` 函数包含一个大的 switch-case，就是实现的这个逻辑。例如，鼠标移动的事件交给 `mouseMoveEvent()` 函数处理，键盘按下的事件交给 `keyPressEvent()` 函数处理。

```c++
bool QWidget::event(QEvent *event)
{
    switch (event->type()) 
    {
    case QEvent::MouseMove:
        mouseMoveEvent((QMouseEvent*)event);
        break;
    case QEvent::MouseButtonPress:
        mousePressEvent((QMouseEvent*)event);
        break;
    case QEvent::MouseButtonRelease:
        mouseReleaseEvent((QMouseEvent*)event);
        break;
    case QEvent::MouseButtonDblClick:
        mouseDoubleClickEvent((QMouseEvent*)event);
        break;
    case QEvent::Wheel:
        wheelEvent((QWheelEvent*)event);
        break;
    case QEvent::KeyPress:
        keyPressEvent((QKeyEvent *)event);
        break;
    case QEvent::Paint:
        paintEvent((QPaintEvent*)event);
        break;
    case QEvent::Move:
        moveEvent((QMoveEvent*)event);
        break;
    case QEvent::Resize:
        resizeEvent((QResizeEvent*)event);
        break;
    case QEvent::Close:
        closeEvent((QCloseEvent*)event);
        break;
    case QEvent::Show:
        showEvent((QShowEvent*) event);
        break;
    ......
    default:
        return QObject::event(event);
    }
    return true;
}
```

这些 event handler 函数全部都是虚函数，所以子类可以通过重写这些函数来实现自己想要的处理逻辑。

子类在继承时，如果只是想在原本的基础上扩展一些功能，那么只需要重写这个虚函数，实现我们想要的部分，其他情况则调用基类的实现。如果完全不调用基类的实现，相当于替换了整个原有的功能。

例如，MyCheckBox 继承自 QCheckBox，想要特殊处理鼠标左键单击事件，那么大概的逻辑如下，

```c++
void MyCheckBox::mousePressEvent(QMouseEvent *event)
{
    if (event->button() == Qt::LeftButton) {
        // handle left mouse button here
    } else {
        // pass on other buttons to base class
        QCheckBox::mousePressEvent(event);
    }
}
```

并不是所有的事件类型都有对应的 event handler，如果需要修改这部分事件的处理，就只能通过重写 `event()` 函数来实现。

## 事件的二次传递

Qt 的事件系统中，有一个 Event Propagation 的概念，姑且翻译成二次传递，它指的是一些界面事件可以向父级传递。

Qt 的界面程序中，控件都是一级一级的，父控件中包含子控件，子控件根据布局被安排在父控件上层显示。例如，一个 QDialog 中有一个 QGroupBox，QGroupBox 中又有两个 QLabel，那么 QDialog 则是这个 QGroupBox 的父控件，QGroupBox 又是这两个 QLabel 的父控件， QDialog 是处于最底层的，QLabel 是处于最上层的。

假设在一个 QLabel 上发生了鼠标移动事件，它首先会被分配给这个 QLabel 进行处理，如果 QLabel 表示不接受这个事件，Qt 的事件分派机制会继续将这个事件发送给父一级的控件，在这个例子中也就是 QGroupBox。除非某个控件表示接受，否则这个事件会一直向父级传递，直到最外层的窗口。

QEvent 的 [accept()](https://doc.qt.io/qt-5/qevent.html#accept) 和 [ignore()](https://doc.qt.io/qt-5/qevent.html#ignore) 函数就是用来表示是否接受这个事件，前者表示接受这个事件，后者表示不接受。这里的是否接受只是逻辑上的，当然你也可以在实际处理某个事件后仍然调用 `ignore()`，以此来将事件传递给父级控件，或者实际不处理但调用 `accept()`，使得父级控件无法收到这个事件。

实际上，只有一部分事件具有这种可以继续传递的特性，它们基本上都是由键盘鼠标操作引发的相关事件，accept 和 ignore 属性也只对这些事件起作用（可以查阅各QEvent派生类的文档来确定是否能二次传递）。在继承重写事件的处理函数时，需要根据实际的功能逻辑，明确调用 `accept()` 或 `ignore()`。

## QCloseEvent

QCloseEvent 的传递有些特殊，它表示关闭事件，也可以理解为关闭的请求。当用户想要关闭某个窗口时，比如点击标题栏上的“X”按钮，就会产生相应的 QCloseEvent 发送给相应的 QWidget 对象。程序中调用 `QWidget::close()` 函数也会产生 QCloseEvent 事件。

QCloseEvent 最终会传递到 `QWidget::closeEvent()` 事件处理函数中。如果接收者允许被关闭，则调用 QCloseEvent 的 `accept()` 函数表示接受关闭请求，随后就会被隐藏，如果接收者拒绝被关闭，则调用 `ingore()` 函数，一切保持原样。`QWidget::closeEvent()` 中的默认实现是接受关闭请求，所以如果程序想进一步处理关闭请求，比如关闭时提示文件未保存，就需要自己重新继承实现 `closeEvent` 函数在其中处理。

## 事件过滤器

有些情况下，可能需要提前截获处理发送给某个对象的事件，Qt 提供了事件过滤器（Event Filter）机制来实现这种需求。任何 QObject 对象都可以作为一个事件过滤器，来过滤发送给其他 QObject 对象的事件。

事件过滤器需要通过 [QObject::installEventFilter(QObject *filterObj)](https://doc.qt.io/qt-5/qobject.html#installEventFilter) 函数来设置，例如，

```c++
A->installEventFilter(B);
```

它表示为对象 A 安装了一个事件过滤器 B。其中 A 和 B 都必须是 QObject 或其子类的对象。过滤器对象 B 需要重写 [bool QObject::eventFilter(QObject *watched, QEvent *event)](https://doc.qt.io/qt-5/qobject.html#eventFilter) 函数来实现事件的过滤处理。任何发送给 A 的事件，在进入 A 的 event 函数之前，都会先经过 B 的 `eventFilter()` 函数。所以在过滤器中，可以提前处理 A 的某些事件。`eventFilter()` 的返回值表示是否截获这个事件，如果返回 true，表示这个事件被过滤器截获了，不会再被发送给A的 `event()` 函数，如果返回 false，这个事件仍然会被 A 的 `event()` 函数收到。

一个过滤器可以同时过滤多个对象的事件，`eventFilter()` 函数的第一个参数即为事件本来的接收者对象。

一个被过滤对象也可以同时安装多个过滤器。这种情况下所有过滤器按照安装的顺序反向依次进行处理，最后安装的过滤器先处理，最先安装的过滤器最后处理。如果其中某个过滤器返回了 true，那么后续的过滤器和被过滤对象都不会收到这个事件。

## 自定义事件

可以通过 Qt 的事件系统来实现自定义事件的分发处理。

自定义事件需要继承 QEvent，并且定义一个类型值，用于表示事件类型，也就是 QEvent 构造函数的唯一参数，和 `QEvent::type()` 函数的返回值。Qt 规定自定义事件的类型值必须大于枚举值 `QEvent::User`，以便区分于 Qt 自身的事件。例如，

```c++
const int MYEVENT_TYPE = QEvent::User + 1;
class MyEvent : public QEvent
{
public:
    MyEvent() : QEvent((QEvent::Type)MYEVENT_TYPE){ }
private:
    //custom data
};
```

发送自定义事件有两种常见的方式，通过 [sendEvent()](https://doc.qt.io/qt-5/qcoreapplication.html#sendEvent) 或者 [postEvent()](https://doc.qt.io/qt-5/qcoreapplication.html#postEvent)。

```c++
bool QCoreApplication::sendEvent(QObject *receiver, QEvent *event)
void QCoreApplication::postEvent(QObject *receiver, QEvent *event, int priority = Qt::NormalEventPriority)
```

`sendEvent()` 会立即将事件发送给接收者receiver对象，并阻塞至接收者处理完成之后再返回。`postEvent()` 则是将事件添加到事件队列中，并立即返回，不等待事件的处理。随后这个事件由事件循环再分发处理。需要注意，使用 `postEvent()` 发送的事件必须是在堆内存上创建的，也就是 new 出来的，而且事件循环分发处理完这个事件后，会自动将其析构掉。

自定义事件最终会经过 `QObject::event()` 函数然后进入到 [QObject::customEvent()](https://doc.qt.io/qt-5/qobject.html#customEvent) 中，所以自定义事件的接收者需要重写这个虚函数来实现处理逻辑。

---

参考：

- https://wiki.qt.io/Threads_Events_QObjects
- https://doc.qt.io/qt-5/eventsandfilters.html
- https://doc.qt.io/archives/qq/qq11-events.html
