## Sink frontends

汇前端是库中提供的汇的一部分，实现了所有汇共享的通用功能。这些功能包括对过滤、异常处理和线程同步的支持。此外，由于格式化对于基于文本的汇来说是典型需求，它也由前端实现。每个汇前端从日志核心接收日志记录，并将它们传递给相关的汇后端。前端不定义如何处理记录，而是定义核心应如何与后端交互。日志记录的处理规则由后端决定。如果你需要创建一种新类型的汇，通常不需要编写自己的前端，因为库中提供了大量前端，涵盖了大多数使用场景。

汇前端继承自汇类模板，日志核心通过该模板提供日志记录。从技术角度讲，可以从汇模板派生出自己的类并实现新的汇，但使用汇前端可以省去大量的日常工作。每个汇前端都与后端关联，在前端构造时也会相应构造后端（除非用户自己提供后端实例），使汇完整。因此，在前端构造后，它可以注册到日志核心中以开始处理记录。有关前端和后端交互的更多详细信息，请参见“汇后端”部分。

以下是汇前端提供的服务的详细概述。

### Basic sink frontend services

所有汇前端都提供了一些基本功能。

#### Filtering
所有汇前端都支持过滤。用户可以指定自定义的过滤函数对象，或者使用库提供的工具构建过滤器。可以通过 `set_filter` 方法设置过滤器，或通过 `reset_filter` 方法清除过滤器。在日志核心调用 `will_consume` 方法时会调用过滤器。如果没有设置过滤器，默认情况下，汇会接受所有日志记录。

---

📓注意

与日志核心一样，所有汇前端假设可以安全地从多个线程并发调用过滤器。对于库提供的过滤器，这没有问题。

---

#### Formatting
对于基于文本的汇后端，前端实现了日志记录的格式化处理。与过滤器类似，可以使用 Lambda 表达式来构建格式化器。可以通过调用 `set_formatter` 方法为基于文本的汇设置格式化器，或者通过 `reset_formatter` 方法清除格式化器。

#### Exception handling
所有汇前端都允许设置异常处理程序，以自定义每个汇的错误处理方式。可以使用 `set_exception_handler` 方法安装一个异常处理函数，当后端处理记录或汇的过滤过程中发生异常时，该函数会在 `catch` 块中被调用且不带参数。异常处理程序可以选择重新抛出异常或将其抑制。在前一种情况下，异常会被传递到日志核心，日志核心中可以有另一层异常处理机制。

---

❗提示

日志核心和日志器也支持安装异常处理程序。

---

库提供了一个方便的工具，可以将异常分派给一元多态函数对象。

---

📓注意

异常处理程序不允许返回值。这意味着一旦发生异常，你不能改变过滤结果，因此过滤将总是失败。

---

📓注意

所有汇前端假设可以安全地从多个线程并发调用异常处理程序。对于库提供的异常分派器，这没有问题。

---

### Unlocked sink frontend

```cpp
#include <boost/log/sinks/unlocked_frontend.hpp>
```

非锁定汇前端使用 `unlocked_sink` 类模板实现。该前端为后端提供了最基本的服务。`unlocked_sink` 前端在访问后端时不进行线程同步，假设后端不需要同步或同步已由后端实现。然而，设置过滤器仍然是线程安全的（即使其他线程通过该汇写入日志，仍可安全地更改 `unlocked_sink` 前端的过滤器）。这是单线程环境中唯一可用的汇前端。使用示例如下：

```cpp
enum severity_level
{
    normal,
    warning,
    error
};

// 一个简单的汇后端，不需要线程同步
class my_backend :
    public sinks::basic_sink_backend< sinks::concurrent_feeding >
{
public:
    // 该函数在每条日志记录写入日志时调用
    void consume(logging::record_view const& rec)
    {
        // 为简洁起见，略去实际的同步代码
        std::cout << rec[expr::smessage] << std::endl;
    }
};

// 完整的汇类型
typedef sinks::unlocked_sink< my_backend > sink_t;

void init_logging()
{
    boost::shared_ptr< logging::core > core = logging::core::get();

    // 最简单的方式，后端是默认构造的
    boost::shared_ptr< sink_t > sink1(new sink_t());
    core->add_sink(sink1);

    // 可以单独构造后端并传递给前端
    boost::shared_ptr< my_backend > backend(new my_backend());
    boost::shared_ptr< sink_t > sink2(new sink_t(backend));
    core->add_sink(sink2);

    // 你可以通过汇接口管理过滤
    sink1->set_filter(expr::attr< severity_level >("Severity") >= warning);
    sink2->set_filter(expr::attr< std::string >("Channel") == "net");
}
```

[完整代码](https://www.boost.org/doc/libs/1_86_0/libs/log/example/doc/sinks_unlocked.cpp)

库提供的所有汇后端都要求前端部分进行线程同步。如果我们尝试在需要更严格线程保证的后端上实例化前端，代码将无法编译。因此，该前端主要用于单线程环境和自定义后端。

### Synchronous sink frontend

```cpp
#include <boost/log/sinks/sync_frontend.hpp>
```

同步汇前端使用 `synchronous_sink` 类模板实现。它类似于 `unlocked_sink`，但在将日志记录传递给后端之前，额外提供了基于互斥锁的线程同步。目前，所有支持格式化的汇后端都要求前端进行线程同步。

同步汇还提供了获取锁定后端指针的功能。只要该指针存在，后端就不会被其他线程访问，除非通过其他前端或直接引用后端进行访问。如果需要在其他线程写入日志时对汇后端进行更新，该功能会很有用。不过要注意，当后端被锁定时，任何试图向汇写入日志记录的线程都会被阻塞，直到后端被释放。

其用法与 `unlocked_sink` 类似。

```cpp
enum severity_level
{
    normal,
    warning,
    error
};

// 完整的汇类型
typedef sinks::synchronous_sink< sinks::text_ostream_backend > sink_t;

void init_logging()
{
    boost::shared_ptr< logging::core > core = logging::core::get();

    // 创建后端并用流初始化
    boost::shared_ptr< sinks::text_ostream_backend > backend =
        boost::make_shared< sinks::text_ostream_backend >();
    backend->add_stream(
        boost::shared_ptr< std::ostream >(&std::clog, boost::null_deleter()));

    // 包装前端并注册到核心
    boost::shared_ptr< sink_t > sink(new sink_t(backend));
    core->add_sink(sink);

    // 你可以通过汇接口管理过滤和格式化
    sink->set_filter(expr::attr< severity_level >("Severity") >= warning);
    sink->set_formatter
    (
        expr::stream
            << "Level: " << expr::attr< severity_level >("Severity")
            << " Message: " << expr::smessage
    );

    // 你还可以以线程安全的方式管理后端
    {
        sink_t::locked_backend_ptr p = sink->locked_backend();
        p->add_stream(boost::make_shared< std::ofstream >("sample.log"));
    } // 后端在此处被释放
}
```

[完整代码](https://www.boost.org/doc/libs/1_86_0/libs/log/example/doc/sinks_sync.cpp)

### Asynchronous sink frontend

```cpp
#include <boost/log/sinks/async_frontend.hpp>

// 相关头文件
#include <boost/log/sinks/unbounded_fifo_queue.hpp>
#include <boost/log/sinks/unbounded_ordering_queue.hpp>
#include <boost/log/sinks/bounded_fifo_queue.hpp>
#include <boost/log/sinks/bounded_ordering_queue.hpp>
#include <boost/log/sinks/drop_on_overflow.hpp>
#include <boost/log/sinks/block_on_overflow.hpp>
```

异步汇前端通过 `asynchronous_sink` 类模板实现。与同步汇类似，异步汇前端提供了一种同步访问后端的方法。所有日志记录都通过专用线程传递给后端，这使其适用于可能会阻塞较长时间的后端（例如网络和其他硬件设备相关的汇）。前端的内部线程在前端构造时创建，并在其析构时加入（这意味着前端的销毁可能会阻塞）。

---

📓注意  

当前异步汇前端的实现使用记录排队。这在记录发射和其实际处理（例如写入文件）之间引入了一定的延迟。这种行为在某些上下文中可能不适合，例如调试易崩溃的应用程序。

---

```cpp
enum severity_level
{
    normal,
    warning,
    error
};

// 完整的汇类型
typedef sinks::asynchronous_sink< sinks::text_ostream_backend > sink_t;

boost::shared_ptr< sink_t > init_logging()
{
    boost::shared_ptr< logging::core > core = logging::core::get();

    // 创建后端并用流初始化
    boost::shared_ptr< sinks::text_ostream_backend > backend =
        boost::make_shared< sinks::text_ostream_backend >();
    backend->add_stream(
        boost::shared_ptr< std::ostream >(&std::clog, boost::null_deleter()));

    // 包装前端并注册到核心
    boost::shared_ptr< sink_t > sink(new sink_t(backend));
    core->add_sink(sink);

    // 你可以通过汇接口管理过滤和格式化
    sink->set_filter(expr::attr< severity_level >("Severity") >= warning);
    sink->set_formatter
    (
        expr::stream
            << "Level: " << expr::attr< severity_level >("Severity")
            << " Message: " << expr::message
    );

    // 你还可以以线程安全的方式管理后端
    {
        sink_t::locked_backend_ptr p = sink->locked_backend();
        p->add_stream(boost::make_shared< std::ofstream >("sample.log"));
    } // 后端在此处被释放

    return sink;
}
```

---

❗重要

如果在多模块应用程序中使用异步日志记录，应该仔细决定何时卸载写入日志的动态加载模块。库的许多地方可能会使用驻留在动态加载模块中的资源。例如，虚拟表、字符串文字和函数。如果这些资源在库使用时被卸载，则应用程序很可能会崩溃。严格来说，这个问题存在于任何汇类型中（并不限于汇），但异步汇引入了额外的问题。无法确定异步汇使用了哪些资源，因为它在专用线程中工作并使用缓冲的日志记录。对此问题没有通用解决方案。建议用户要么在应用程序运行期间避免动态模块卸载，要么避免异步日志记录。作为解决问题的附加方法，可以在卸载任何模块之前尝试停止所有异步汇，卸载后再重新创建它们。然而，避免动态卸载是完全解决问题的唯一方法。

---

为了停止专用线程向后端传递日志记录，可以调用前端的 `stop` 方法。该方法会在前端析构时自动调用。`stop` 方法与线程中断不同，只会在后端处理正在传递的日志记录时终止传递循环（即它不会中断已经开始的记录处理）。但是，在返回 `stop` 方法后，可能仍有一些记录留在队列中。为了将它们刷新到后端，必须额外调用 `feed_records` 方法。这在应用程序终止阶段是有用的。

```cpp
void stop_logging(boost::shared_ptr< sink_t >& sink)
{
    boost::shared_ptr< logging::core > core = logging::core::get();

    // 从核心中移除汇，以便不再向其传递记录
    core->remove_sink(sink);

    // 终止传递循环
    sink->stop();

    // 刷新所有可能留下的缓冲日志记录
    sink->flush();

    sink.reset();
}
```

[完整代码](https://www.boost.org/doc/libs/1_86_0/libs/log/example/doc/sinks_async.cpp)

为日志记录传递专用线程的创建可以通过前端的可选布尔参数 `start_thread` 来抑制。在这种情况下，用户可以选择处理记录的方式：

- 调用前端的 `run` 方法。该调用将在传递循环中阻塞。可以通过调用 `stop` 来中断此循环。
- 定期调用 `feed_records`。该方法将处理在调用时前端队列中的所有日志记录，然后返回。

---

📓注意

用户应小心不要同时混合这两种方法。同时，如果专用传递线程正在运行（即在构造时没有指定 `start_thread`，或者其值为 `true`），则不应调用这些方法。

---

#### Customizing record queueing strategy
`asynchronous_sink` 类模板可以使用记录排队策略进行自定义。库提供了几种策略：

- `unbounded_fifo_queue`：这种策略是默认的。如其名所示，队列的深度不受限制，并且不对日志记录进行排序。
- `unbounded_ordering_queue`：与 `unbounded_fifo_queue` 类似，队列没有深度限制，但对排队的记录应用顺序。
- `bounded_fifo_queue`：队列的深度有限，由模板参数指定，并且应用溢出处理策略。不应用记录排序。
- `bounded_ordering_queue`：类似于 `bounded_fifo_queue`，但还应用日志记录排序。

---

❗警告

请注意无限制排队策略。由于队列的深度不受限制，如果日志记录的生成速度持续超过后端处理速度，队列将无节制地增长，表现为内存泄漏。

---

有限队列支持以下溢出策略：

- `drop_on_overflow`：当队列满时，静默丢弃过多的日志记录。
- `block_on_overflow`：当队列满时，阻塞日志记录线程，直到后端传递线程处理了一些排队记录。

例如，以下是如何修改之前的示例以将记录队列限制为 100 个元素：

```cpp
// 完整的汇类型
typedef sinks::asynchronous_sink<
    sinks::text_ostream_backend,
    sinks::bounded_fifo_queue<          // 日志记录排队策略
        100,                            // 记录队列容量
        sinks::drop_on_overflow         // 溢出处理策略
    >
> sink_t;

boost::shared_ptr< sink_t > init_logging()
{
    boost::shared_ptr< logging::core > core = logging::core::get();

    // 创建后端并用流初始化
    boost::shared_ptr< sinks::text_ostream_backend > backend =
        boost::make_shared< sinks::text_ostream_backend >();
    backend->add_stream(
        boost::shared_ptr< std::ostream >(&std::clog, boost::null_deleter()));

    // 包装前端并注册到核心
    boost::shared_ptr< sink_t > sink(new sink_t(backend));
    core->add_sink(sink);

    // ...

    return sink;
}
```

[完整代码](https://www.boost.org/doc/libs/1_86_0/libs/log/example/doc/sinks_async_bounded.cpp)

还可以在库分发中查看 `bounded_async_log` 示例。

#### Ordering log records
日志记录排序对于缓解多线程应用程序中存在的弱记录排序问题非常有用。

排序排队策略会在记录处理过程中引入小的延迟。可以在前端构造时指定延迟时间和排序谓词。使用日志记录排序工具实现排序谓词可能会很有用。

```cpp
// 完整的汇类型
typedef sinks::asynchronous_sink<
    sinks::text_ostream_backend,
    sinks::unbounded_ordering_queue<                                             // 日志记录排队策略
        logging::attribute_value_ordering<                                       // 日志记录排序谓词类型
            unsigned int,                                                        // 属性值类型
            std::less< unsigned int >                                            // 可选的属性值比较谓词；默认使用 `std::less`
        >
    >
> sink_t;

boost::shared_ptr< sink_t > init_logging()
{
    boost::shared_ptr< logging::core > core = logging::core::get();

    // 创建后端并用流初始化
    boost::shared_ptr< sinks::text_ostream_backend > backend =
        boost::make_shared< sinks::text_ostream_backend >();
    backend->add_stream(
        boost::shared_ptr< std::ostream >(&std::clog, boost::null_deleter()));

    // 包装前端并注册到核心
    boost::shared_ptr< sink_t > sink(new sink_t(
        backend,                                                                 // 指向预初始化后端的指针
        keywords::order = logging::make_attr_ordering< unsigned int >(           // 日志记录排序谓词
            "LineID", std::less< unsigned int >()),
        keywords::ordering_window = boost::posix_time::seconds(1)                // 日志记录处理的延迟时间
    ));
    core->add_sink(sink);

    // 你可以通过汇接口管理过滤和格式化
    sink->set_filter(expr::attr< severity_level >("Severity") >= warning);
    sink->set_formatter
    (
        expr::stream
            << "Level: " << expr::attr< severity_level >("Severity")
            << " Message: " << expr::smessage
    );

    // 你还可以以线程安全的方式管理后端
    {
        sink_t::locked_backend_ptr p = sink->locked_backend();
        p->add_stream(boost::make_shared< std::ofstream >("sample.log"));
    } // 后端在此处被释放

    return sink;
}
```

在上述代码示例中，汇前端将在内部队列中保留日志记录长达一秒，并根据类型为 `unsigned int` 的日志记录计数器应用排序。`ordering_window` 参数是可选的，默认为一些合理的小系统特定值，这将足以保持日志记录的时间顺序流向后端。

即使在停止内部传递循环时，前端仍会维护排序窗口，因此可以在不破坏记录排序的情况下重新进入循环。另一方面，为了确保所有日志记录被刷新到后端，必须在应用程序结束时调用 `flush` 方法。

```cpp
void stop_logging(boost::shared_ptr< sink_t >& sink)
{
    boost::shared_ptr< logging::core > core = logging::core::get();

    // 从核心中移除汇，以便不再向其传递记录
    core->remove_sink(sink);

    // 终止传递循环
    sink->stop();

    // 刷新所有可能留下的缓冲日志记录
    sink->flush();

    sink.reset();
}
```

此技术也在库分发中的 `async_log` 示例中演示。