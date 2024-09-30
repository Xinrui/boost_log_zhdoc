## Logging sources

### Basic loggers
```cpp
#include <boost/log/sources/basic_logger.hpp>
```
库提供的最简单的日志源是日志记录器 `logger` 及其线程安全版本 `logger_mt`（对应的宽字符记录器为 `wlogger` 和 `wlogger_mt`）。这些日志记录器仅提供在其内部存储源特定属性的能力，并且当然也可以形成日志记录。此类型的记录器应该在不需要高级功能（如严重性级别检查）时使用。它也可以作为收集应用程序统计信息和注册应用程序事件的工具，例如通知和警报。在这种情况下，日志记录器通常与作用域属性结合使用，以将所需数据附加到通知事件。以下是一个使用示例：

```cpp
class network_connection
{
    src::logger m_logger;
    logging::attribute_set::iterator m_remote_addr;

public:
    void on_connected(std::string const& remote_addr)
    {
        // 将远程地址放入记录器，以便自动附加到通过记录器写入的每条日志记录
        m_remote_addr = m_logger.add_attribute("RemoteAddress",
            attrs::constant< std::string >(remote_addr)).first;

        // 直接的日志记录方式
        if (logging::record rec = m_logger.open_record())
        {
            rec.attribute_values().insert("Message",
                attrs::make_attribute_value(std::string("Connection established")));
            m_logger.push_record(boost::move(rec));
        }
    }
    void on_disconnected()
    {
        // 更简单的日志记录方式：上面的 "if" 条件被封装在一个整洁的宏中
        BOOST_LOG(m_logger) << "Connection shut down";

        // 移除远程地址的属性
        m_logger.remove_attribute(m_remote_addr);
    }
    void on_data_received(std::size_t size)
    {
        // 将大小作为额外属性放入
        // 以便稍后如果需要可以收集和累积。
        // 该属性将仅附加到此日志记录。
        BOOST_LOG(m_logger) << logging::add_value("ReceivedSize", size) << "Some data received";
    }
    void on_data_sent(std::size_t size)
    {
        BOOST_LOG(m_logger) << logging::add_value("SentSize", size) << "Some data sent";
    }
};
```

上面的代码片段中的 `network_connection` 类表示在网络相关应用程序中实现简单日志记录和统计信息收集的一种方法。该类的每个方法有效地标记一个可以在接收端级别跟踪和收集的相应事件。此外，该类的其他方法（为简化起见未在此处显示）也能够写入日志。请注意，在 `network_connection` 对象的连接状态下所做的每条日志记录都会隐式标记远程站点的地址。

### Loggers with severity level support
```cpp
#include <boost/log/sources/severity_feature.hpp>
#include <boost/log/sources/severity_logger.hpp>
```
根据某种严重性或重要性级别区分日志记录的能力是最常被请求的功能之一。类模板 `severity_logger` 和 `severity_logger_mt`（以及它们的宽字符对应 `wseverity_logger` 和 `wseverity_logger_mt`）提供了此功能。

日志记录器自动注册一个名为“Severity”的特殊源特定属性，该属性可以以紧凑高效的方式为每条记录设置，通过传递一个命名参数 `severity` 到构造函数和/或 `open_record` 方法。如果传递给记录器构造函数，则 `severity` 参数设置将使用的默认严重性级别（如果在 `open_record` 参数中未提供）。传递给 `open_record` 方法的 `severity` 参数设置特定日志记录的级别。严重性级别的类型可以作为模板参数提供给记录器类模板。默认类型为 `int`。

此属性的实际值及其含义完全由用户定义。然而，建议将值为零的级别作为其他值的基准。这是因为默认构造的记录器对象将其默认严重性级别设置为零。还建议在整个应用程序中定义相同的严重性级别，以避免后续日志书写的混淆。以下代码片段展示了 `severity_logger` 的使用：

```cpp
// 定义自己的严重性级别
enum severity_level
{
    normal,
    notification,
    warning,
    error,
    critical
};

void logging_function()
{
    // 记录器在构造时隐式添加一个类型为 'severity_level' 的源特定属性 'Severity'
    src::severity_logger< severity_level > slg;

    BOOST_LOG_SEV(slg, normal) << "A regular message";
    BOOST_LOG_SEV(slg, warning) << "Something bad is going on but I can handle it";
    BOOST_LOG_SEV(slg, critical) << "Everything crumbles, shoot me now!";
}

void default_severity()
{
    // 默认严重性可以在构造函数中指定。
    src::severity_logger< severity_level > error_lg(keywords::severity = error);

    BOOST_LOG(error_lg) << "An error level log record (by default)";

    // 显式指定的级别会覆盖默认值
    BOOST_LOG_SEV(error_lg, warning) << "A warning level log record (overrode the default)";
}
```

或者，如果你喜欢没有宏的日志记录：

```cpp
void manual_logging()
{
    src::severity_logger< severity_level > slg;

    logging::record rec = slg.open_record(keywords::severity = normal);
    if (rec)
    {
        logging::record_ostream strm(rec);
        strm << "A regular message";
        strm.flush();
        slg.push_record(boost::move(rec));
    }
}
```

当然，严重性记录器还提供与[基本记录器](#basic-loggers)相同的功能。

### Loggers with channel support
```cpp
#include <boost/log/sources/channel_feature.hpp>
#include <boost/log/sources/channel_logger.hpp>
```
有时，重要的是将日志记录与某个应用程序组件关联，例如模块或类名，记录信息与某个特定功能领域（例如网络或文件系统相关消息）的关系，或者某个任意标签，可用于稍后将这些记录路由到特定接收端。此功能通过 `channel_logger`、`channel_logger_mt` 及其宽字符对应的 `wchannel_logger`、`wchannel_logger_mt` 实现。这些日志记录器自动注册一个名为“Channel”的属性。默认通道名称可以在记录器构造函数中通过命名参数 `channel` 设置。通道属性值的类型可以作为模板参数提供给记录器，默认情况下为 `std::string`（对于宽字符记录器则为 `std::wstring`）。此外，使用方式类似于[基本日志记录器](#basic-loggers)：

```cpp
class network_connection
{
    src::channel_logger< > m_net, m_stat;
    logging::attribute_set::iterator m_net_remote_addr, m_stat_remote_addr;

public:
    network_connection() :
        // 我们可以通过此记录器转储网络相关消息
        // 并能够稍后对其进行过滤
        m_net(keywords::channel = "net"),
        // 我们还可以将统计记录分隔到不同的通道
        // 以将其路由到不同的接收端
        m_stat(keywords::channel = "stat")
    {
    }

    void on_connected(std::string const& remote_addr)
    {
        // 将远程地址添加到两个通道
        attrs::constant< std::string > addr(remote_addr);
        m_net_remote_addr = m_net.add_attribute("RemoteAddress", addr).first;
        m_stat_remote_addr = m_stat.add_attribute("RemoteAddress", addr).first;

        // 将消息放入“net”通道
        BOOST_LOG(m_net) << "Connection established";
    }

    void on_disconnected()
    {
        // 将消息放入“net”通道
        BOOST_LOG(m_net) << "Connection shut down";

        // 移除远程地址的属性
        m_net.remove_attribute(m_net_remote_addr);
        m_stat.remove_attribute(m_stat_remote_addr);
    }

    void on_data_received(std::size_t size)
    {
        BOOST_LOG(m_stat) << logging::add_value("ReceivedSize", size) << "Some data received";
    }

    void on_data_sent(std::size_t size)
    {
        BOOST_LOG(m_stat) << logging::add_value("SentSize", size) << "Some data sent";
    }
};
```

还可以为单个日志记录设置通道名称。当使用全局记录器而不是特定对象的记录器时，这可能很有用。可以通过在记录器上调用通道修饰符或使用日志记录的特殊宏来设置通道名称。例如：

```cpp
// 定义一个全局记录器
BOOST_LOG_INLINE_GLOBAL_LOGGER_CTOR_ARGS(my_logger, src::channel_logger_mt< >, (keywords::channel = "general"))

class network_connection
{
    std::string m_remote_addr;

public:
    void on_connected(std::string const& remote_addr)
    {
        m_remote_addr = remote_addr;

        // 将消息放入“net”通道
        BOOST

_LOG(my_logger) << "Connection established";
    }

    void on_disconnected()
    {
        // 将消息放入“net”通道
        BOOST_LOG(my_logger) << "Connection shut down";
    }
};
```
```cpp
// 定义一个全局日志记录器
BOOST_LOG_INLINE_GLOBAL_LOGGER_CTOR_ARGS(my_logger, src::channel_logger_mt< >, (keywords::channel = "general"))

class network_connection
{
    std::string m_remote_addr;

public:
    void on_connected(std::string const& remote_addr)
    {
        m_remote_addr = remote_addr;

        // 将消息放入“net”通道
        BOOST_LOG_CHANNEL(my_logger::get(), "net")
            << logging::add_value("RemoteAddress", m_remote_addr)
            << "连接已建立";
    }

    void on_disconnected()
    {
        // 将消息放入“net”通道
        BOOST_LOG_CHANNEL(my_logger::get(), "net")
            << logging::add_value("RemoteAddress", m_remote_addr)
            << "连接已关闭";

        m_remote_addr.clear();
    }

    void on_data_received(std::size_t size)
    {
        BOOST_LOG_CHANNEL(my_logger::get(), "stat")
            << logging::add_value("RemoteAddress", m_remote_addr)
            << logging::add_value("ReceivedSize", size)
            << "接收到一些数据";
    }

    void on_data_sent(std::size_t size)
    {
        BOOST_LOG_CHANNEL(my_logger::get(), "stat")
            << logging::add_value("RemoteAddress", m_remote_addr)
            << logging::add_value("SentSize", size)
            << "发送了一些数据";
    }
};
```

请注意，改变通道名称是持久的，因此除非通道名称被重置，否则后续记录也将属于新通道。

---

> ❗提示
> 
> 出于性能考虑，建议尽可能避免为每条日志记录动态设置通道名称。改变通道名称涉及动态内存分配。为不同通道使用不同的日志记录器可以避免这种开销。

--- 

### Loggers with exception handling support
```cpp
#include <boost/log/sources/exception_handler_feature.hpp>
```

该库提供了一种日志记录器功能，使用户能够在日志记录器级别处理或抑制异常。`exception_handler` 功能为日志记录器添加了一个 `set_exception_handler` 方法，该方法允许设置一个函数对象，以便在日志核心在过滤或处理日志记录时抛出异常时调用。可以使用库提供的适配器来简化异常处理器的实现。用法示例如下：

```cpp
enum severity_level
{
    normal,
    warning,
    error
};

// 一个允许拦截异常并支持严重级别的日志记录器类
class my_logger_mt :
    public src::basic_composite_logger<
        char,
        my_logger_mt,
        src::multi_thread_model< boost::shared_mutex >,
        src::features<
            src::severity< severity_level >,
            src::exception_handler
        >
    >
{
    BOOST_LOG_FORWARD_LOGGER_MEMBERS(my_logger_mt)
};

BOOST_LOG_INLINE_GLOBAL_LOGGER_INIT(my_logger, my_logger_mt)
{
    my_logger_mt lg;

    // 设置异常处理器：通过此日志记录器记录时发生的所有异常都将被抑制
    lg.set_exception_handler(logging::make_exception_suppressor());

    return lg;
}

void logging_function()
{
    // 这不会抛出异常
    BOOST_LOG_SEV(my_logger::get(), normal) << "Hello, world";
}
```

---

❗提示

日志核心和接收器前端也支持安装异常处理器。

---

### Loggers with mixed features

```cpp
#include <boost/log/sources/severity_channel_logger.hpp>
```

如果你想知道是否可以在一个日志记录器中使用多个混合特性，那么答案是肯定的。该库提供了 `severity_channel_logger` 和 `severity_channel_logger_mt`（以及它们的宽字符版本 `wseverity_channel_logger` 和 `wseverity_channel_logger_mt`），这些日志记录器结合了日志严重级别和通道支持的特性。这些复合日志记录器也是模板，这允许你指定严重级别和通道类型。你还可以设计自己的日志特性，并将它们与库提供的特性结合使用，具体描述可以参考“扩展库”部分。

使用带有多个特性的日志记录器在概念上与使用单一特性的日志记录器并无不同。例如，以下是如何使用 `severity_channel_logger_mt`：

```cpp
enum severity_level
{
    normal,
    notification,
    warning,
    error,
    critical
};

typedef src::severity_channel_logger_mt<
    severity_level,     // 严重级别的类型
    std::string         // 通道名称的类型
> my_logger_mt;


BOOST_LOG_INLINE_GLOBAL_LOGGER_INIT(my_logger, my_logger_mt)
{
    // 在构造时指定通道名称，类似于 channel_logger 的用法
    return my_logger_mt(keywords::channel = "my_logger");
}

void logging_function()
{
    // 使用严重级别进行日志记录。记录将同时包含严重级别和通道名称。
    BOOST_LOG_SEV(my_logger::get(), normal) << "Hello, world!";
}
```

通过这种方式，你可以在日志记录器中同时使用严重级别和通道功能，轻松实现多特性日志记录。

### Global storage for loggers

```cpp
#include <boost/log/sources/global_logger_storage.hpp>
```

有时在编写日志时拥有一个日志记录器对象会显得不便。这个问题在函数式风格的代码中尤为明显，因为没有明显的地方可以存储日志记录器。另一个常见问题是通用库也需要日志记录功能。在这种情况下，拥有一个或多个全局日志记录器会更加方便，可以随时在任何地方访问它们。`std::cout` 就是一个类似的全局日志记录器的例子。

该库提供了一种声明全局日志记录器的方法，可以像 `std::cout` 一样访问它们。实际上，这个功能可以用于任何日志记录器，包括用户自定义的日志记录器。声明了全局日志记录器后，您可以确保从应用程序代码的任何地方对该日志记录器实例进行线程安全的访问。该库还保证，即使跨模块边界，全局日志记录器实例也将是唯一的。这使得在仅包含头文件的组件中使用日志记录成为可能，这些组件可能被编译到不同的模块中。

有人可能会想，为什么需要特殊的方式来创建全局日志记录器？为什么不直接在命名空间范围内声明一个日志记录器变量并在需要时使用它？虽然从技术上讲这是可行的，但在以下几个原因下声明和使用全局日志记录器变量是复杂的：

1. **命名空间范围变量的初始化顺序并没有在 C++ 标准中规定**。这意味着通常无法在初始化阶段（即 `main` 函数之前）使用日志记录器。
2. **命名空间范围变量的初始化不是线程安全的**。你可能会遇到同一个日志记录器被初始化两次，或者在日志记录器未初始化时使用它。
3. **在仅包含头文件的库中使用命名空间范围变量相当复杂**。要么声明具有外部链接的变量，并仅在单个翻译单元中定义它（即在单独的 .cpp 文件中定义，这违背了“头文件库”的理念），要么定义具有内部链接的变量，或者在匿名命名空间中定义特殊情况（这很可能会破坏 ODR（一个定义规则），并在不同翻译单元中使用该头文件时产生意外结果）。还有一些编译器特定和标准的技巧来解决这个问题，但它们既不简单也不具有可移植性。
4. **在大多数平台上，命名空间范围变量是局限于其编译所在的模块的**。即，如果变量 `a` 具有外部链接并被编译到模块 X 和 Y 中，这些模块中的每个都有自己的 `a` 变量副本。更糟糕的是，在其他平台上，这个变量可能会在模块之间共享。

全局日志记录器存储旨在消除所有这些问题。

声明全局日志记录器的最简单方法是使用以下宏：

```cpp
BOOST_LOG_INLINE_GLOBAL_LOGGER_DEFAULT(my_logger, src::severity_logger_mt< >)
```

`my_logger` 参数为日志记录器命名，它可用于获取日志记录器实例。该名称作为声明的日志记录器的标记。第二个参数表示日志记录器的类型。在多线程应用程序中，当日志记录器可以从不同线程访问时，用户通常会使用线程安全版本的日志记录器。

如果需要向日志记录器构造函数传递参数，可以使用另一个宏：

```cpp
BOOST_LOG_INLINE_GLOBAL_LOGGER_CTOR_ARGS(
    my_logger,
    src::severity_channel_logger< >,
    (keywords::severity = error)(keywords::channel = "my_channel"))
```

最后一个宏参数是传递给日志记录器构造函数的 [Boost.Preprocessor](https://www.boost.org/doc/libs/release/libs/preprocessor/doc/index.html) 参数序列。不过在使用非常量表达式和对对象的引用作为构造函数参数时要小心，因为参数只会被评估一次，并且通常很难确定评估发生的确切时刻。日志记录器是在应用程序第一次请求时构建的，由了解日志记录器声明的部分负责调用。用户必须确保此时所有参数处于有效状态。

第三个宏提供了最大的初始化灵活性，允许用户实际定义创建日志记录器的逻辑：

```cpp
BOOST_LOG_INLINE_GLOBAL_LOGGER_INIT(my_logger, src::severity_logger_mt)
{
    // 执行日志记录器初始化时需要完成的操作，
    // 例如添加一个秒表属性。
    src::severity_logger_mt< > lg;
    lg.add_attribute("StopWatch", attrs::timer());
    // 初始化例程必须返回日志记录器实例
    return lg;
}
```

与 `BOOST_LOG_INLINE_GLOBAL_LOGGER_CTOR_ARGS` 宏类似，初始化代码只在第一次请求日志记录器时调用一次。

---

⚠重要提示

请注意**单一定义规则**（ODR）问题。无论选择哪种日志记录器声明方式，<u>都应确保在所有出现的地方都以完全相同的方式声明日志记录器</u>，并且<u>声明中涉及的所有符号名称都解析为相同的实体</u>。后者包括在 `BOOST_LOG_INLINE_GLOBAL_LOGGER_INIT` 宏的初始化例程中使用的外部变量、函数和类型的名称。该库在一定程度上试图防止 ODR 违规，但一般情况下，如果违反了规则，行为是未定义的。

---

为了缓解 ODR 问题，可以将日志记录器声明与其初始化例程分开。该库提供了以下宏来实现这一目的：

- `BOOST_LOG_GLOBAL_LOGGER` 提供日志记录器声明。它可以在头文件中使用，类似于上述 `BOOST_LOG_INLINE_GLOBAL_LOGGER*` 宏。
- `BOOST_LOG_GLOBAL_LOGGER_INIT`、`BOOST_LOG_GLOBAL_LOGGER_DEFAULT` 和 `BOOST_LOG_GLOBAL_LOGGER_CTOR_ARGS` 定义日志记录器的初始化例程。它们的语义和用法与相应的 `BOOST_LOG_INLINE_GLOBAL_LOGGER*` 宏相似，唯一的例外是：这些宏应在单个 .cpp 文件中使用。

例如：

```cpp
// my_logger.h
// ===========

BOOST_LOG_GLOBAL_LOGGER(my_logger, src::severity_logger_mt)


// my_logger.cpp
// =============

#include "my_logger.h"

BOOST_LOG_GLOBAL_LOGGER_INIT(my_logger, src::severity_logger_mt)
{
    src::severity_logger_mt< > lg;
    lg.add_attribute("StopWatch", attrs::timer());
    return lg;
}
```

无论你使用哪个宏声明日志记录器，你都可以通过日志记录器标记的静态 `get` 函数获取日志记录器实例：

```cpp
src::severity_logger_mt< >& lg = my_logger::get();
```

进一步使用该日志记录器时，与使用相应类型的常规日志记录器对象相同。

---

❗警告

请注意，不建议在应用程序的去初始化阶段使用全局日志记录器。像应用程序中的其他全局对象一样，全局日志记录器在你使用它之前可能已经被销毁。在这种情况下，最好有一个专用的日志记录器对象，确保其在需要时可用。

---