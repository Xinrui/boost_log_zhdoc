## Setting up sinks

有时，简单的日志记录无法提供足够的灵活性。例如，用户可能希望使用更复杂的日志处理逻辑，而不仅仅是将日志打印到控制台。为此，您需要构建日志汇并将其注册到日志核心。这通常只需在应用程序的启动代码中执行一次。

---

**📓注意**

必须提到，在前面的部分中，我们并没有初始化任何汇，而简单的日志记录仍然可以正常工作。这是因为库中包含一个默认汇，当用户未设置任何汇时，它会作为回退使用。这个汇始终以固定格式将日志记录打印到控制台，这在我们之前的示例中已经看到。默认汇主要是为了允许立即使用简单日志记录，而无需进行任何库初始化。一旦您向日志核心添加了任何汇，默认汇将不再被使用。不过，您仍然可以使用简单日志记录宏。

---

### File logging unleashed

作为一个起点，这里是如何初始化文件日志记录的示例：

```cpp
void init()
{
    logging::add_file_log("sample.log");

    logging::core::get()->set_filter
    (
        logging::trivial::severity >= logging::trivial::info
    );
}
```

新增的部分是调用 `add_file_log` 函数。顾名思义，该函数初始化一个将日志记录存储到文本文件中的汇。该函数还接受多个自定义选项，例如文件轮换间隔和大小限制。例如：

```cpp
void init()
{
    logging::add_file_log
    (
        keywords::file_name = "sample_%N.log",                                        // file name pattern
        keywords::rotation_size = 10 * 1024 * 1024,                                   // rotate files every 10 MiB...
        keywords::time_based_rotation = sinks::file::rotation_at_time_point(0, 0, 0), // ...or at midnight
        keywords::format = "[%TimeStamp%]: %Message%"                                 // log record format
    );

    logging::core::get()->set_filter
    (
        logging::trivial::severity >= logging::trivial::info
    );
}
```

[完整代码](https://www.boost.org/doc/libs/1_86_0/libs/log/example/doc/tutorial_file.cpp)

您可以看到，选项以命名的方式传递给函数。这种方法在库的许多其他地方也被采用，您会习惯它。参数的含义大多是自解释的，并在本手册中有文档说明（有关文本文件汇的说明请参见此处）。本节描述了此和其他便捷的初始化函数。

---

❗提示

您可以注册多个汇。每个汇将独立接收和处理您发出的日志记录，彼此之间不受影响。

---

### Sinks in depth: More sinks

如果您不想深入了解细节，可以跳过本节并继续阅读下一节。否则，如果您需要对汇配置进行更全面的控制，或者想使用比帮助函数提供的更多汇，您可以手动注册汇。

最简单的形式，上一节中调用的 `add_file_log` 函数几乎等同于以下代码：

```cpp
void init()
{
    // 构造日志汇
    typedef sinks::synchronous_sink< sinks::text_ostream_backend > text_sink;
    boost::shared_ptr< text_sink > sink = boost::make_shared< text_sink >();

    // 添加一个流，将日志写入文件
    sink->locked_backend()->add_stream(
        boost::make_shared< std::ofstream >("sample.log"));

    // 将汇注册到日志核心
    logging::core::get()->add_sink(sink);
}
```

[完整代码](https://www.boost.org/doc/libs/1_86_0/libs/log/example/doc/tutorial_file_manual.cpp)

好的，您可能已经注意到，汇由两个类组成：前端和后端。前端（在上面的代码片段中是 `synchronous_sink` 类模板）负责所有汇的常见任务，例如线程同步模型、过滤和文本汇的格式化。后端（上面的 `text_ostream_backend` 类）实现与汇特定的功能，例如在这种情况下写入文件。库提供了许多可以直接配合使用的前端和后端。

上面的 `synchronous_sink` 类模板表明汇是同步的，这意味着多个线程可以同时记录日志，并且在竞争时会阻塞。这意味着后端 `text_ostream_backend` 完全不必担心多线程问题。还有其他可用的汇前端，您可以在[此处](./../detailed_features_description/sink_frontends.md)了解更多信息。

`text_ostream_backend` 类将格式化的日志记录写入与标准库兼容的流。我们在上面使用了文件流，但也可以使用任何类型的流。例如，将输出添加到控制台可以如下实现：

```cpp
#include <boost/core/null_deleter.hpp>

// 我们必须提供一个空的删除器，以避免销毁全局的流对象
boost::shared_ptr< std::ostream > stream(&std::clog, boost::null_deleter());
sink->locked_backend()->add_stream(stream);
```

`text_ostream_backend` 支持添加多个流。在这种情况下，它的输出会被复制到所有添加的流中。这对于同时将输出复制到控制台和文件非常有用，因为所有的过滤、格式化和其他库的开销仅在每条记录的汇中执行一次。

---

❗提示
 
请注意，注册多个独立的汇和注册一个具有多个目标流的汇之间的区别。前者允许对每个汇进行独立的输出定制，而后者在不需要此类定制的情况下会显著提高速度。此功能特定于这个特定的后端。

---

该库提供了多种后端，提供不同的日志处理逻辑。例如，通过指定 syslog 后端，可以将日志记录通过网络发送到 syslog 服务器；或者通过设置 Windows NT 事件日志后端，可以使用标准 Windows 工具监控应用程序运行时。

最后值得注意的是，`locked_backend` 成员函数用于访问汇后端。它用于获取对后端的线程安全独占访问，并由所有汇前端提供。该函数返回一个智能指针，只要它存在，后端就被锁定（这意味着即使另一个线程尝试记录日志并将日志记录传递给汇，也不会被记录，直到释放后端）。唯一的例外是 `unlocked_sink` 前端，它根本不进行同步，仅返回一个解锁的指针指向后端。
