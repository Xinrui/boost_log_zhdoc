## Log record formatting

如果您尝试运行前面章节中的示例，您可能会注意到只有日志记录消息被写入文件。这是库在未设置格式化器时的默认行为。即使您向日志核心或记录器添加了属性，属性值也不会到达输出，除非您指定一个将使用这些值的格式化器。

回到之前教程部分的一个示例：

```cpp
void init()
{
    logging::add_file_log
    (
        keywords::file_name = "sample_%N.log",
        keywords::rotation_size = 10 * 1024 * 1024,
        keywords::time_based_rotation = sinks::file::rotation_at_time_point(0, 0, 0),
        keywords::format = "[%TimeStamp%]: %Message%"
    );

    logging::core::get()->set_filter
    (
        logging::trivial::severity >= logging::trivial::info
    );
}
```

在 `add_file_log` 函数中，`format` 参数允许指定日志记录的格式。如果您更喜欢手动设置汇，汇前端提供了 `set_formatter` 成员函数用于此目的。

格式可以通过多种方式指定，下面将详细描述。

### Lambda-style formatters
您可以使用类似于 lambda 的表达式创建格式化器，如下所示：

```cpp
void init()
{
    logging::add_file_log
    (
        keywords::file_name = "sample_%N.log",
        // 这将使汇写入看起来像这样的日志记录：
        // 1: <normal> 一条正常严重级别的消息
        // 2: <error> 一条错误严重级别的消息
        keywords::format =
        (
            expr::stream
                << expr::attr< unsigned int >("LineID")
                << ": <" << logging::trivial::severity
                << "> " << expr::smessage
        )
    );
}
```

[完整代码](https://www.boost.org/doc/libs/1_86_0/libs/log/example/doc/tutorial_fmt_stream.cpp)

这里，`stream` 是格式化记录的占位符。其他插入参数，如 `attr` 和 `message`，是定义应该存储在流中的内容的操纵符。我们已经在过滤表达式中看到了严重性占位符，此处它被用于格式化器。这是一个很好的统一：您可以在过滤器和格式化器中使用相同的占位符。`attr` 占位符与 `severity` 占位符类似，因为它也表示属性值。不同之处在于，`severity` 占位符代表名为 "Severity" 且类型为 `trivial::severity_level` 的特定属性，而 `attr` 可用于表示任何属性。否则这两个占位符是等价的。例如，可以用以下内容替换 `severity`：

```cpp
expr::attr< logging::trivial::severity_level >("Severity")
```

---

❗提示

如前一节所示，可以为用户属性定义像 `severity` 这样的占位符。使用这种占位符的额外好处是可以将有关属性的所有信息（名称和值类型）集中在占位符定义中。这使得编码不易出错（您不会拼写错误属性名称或指定错误值类型），因此建议以这种方式定义新属性并在模板表达式中使用它们。

---

还有其他格式化器操纵符提供对日期、时间和其他类型的高级支持。一些操纵符接受其他参数来定制其行为。大多数这些参数都是命名的，可以采用 [Boost.Parameter](https://www.boost.org/doc/libs/release/libs/parameter/doc/html/index.html) 风格传递。

为了更改，我们来看看手动初始化汇时是如何做的：

```cpp
void init()
{
    typedef sinks::synchronous_sink< sinks::text_ostream_backend > text_sink;
    boost::shared_ptr< text_sink > sink = boost::make_shared< text_sink >();

    sink->locked_backend()->add_stream(
        boost::make_shared< std::ofstream >("sample.log"));

    sink->set_formatter
    (
        expr::stream
               // 行 ID 将以十六进制形式写入，8 位，零填充
            << std::hex << std::setw(8) << std::setfill('0') << expr::attr< unsigned int >("LineID")
            << ": <" << logging::trivial::severity
            << "> " << expr::smessage
    );

    logging::core::get()->add_sink(sink);
}
```

[完整代码](https://www.boost.org/doc/libs/1_86_0/libs/log/example/doc/tutorial_fmt_stream_manual.cpp)

可以看到，可以在表达式中绑定格式更改操纵符；这些操纵符会影响日志记录格式化时随后的属性值格式，就像处理流一样。更多操纵符将在详细特性描述部分中描述。

### Boost.Format-style formatters
作为替代，您可以使用与 [Boost.Format](https://www.boost.org/doc/libs/release/libs/format/index.html) 类似的语法定义格式化器。上述描述的相同格式化器可以如下编写：

```cpp
void init()
{
    typedef sinks::synchronous_sink< sinks::text_ostream_backend > text_sink;
    boost::shared_ptr< text_sink > sink = boost::make_shared< text_sink >();

    sink->locked_backend()->add_stream(
        boost::make_shared< std::ofstream >("sample.log"));

    // 这将使汇写入看起来像这样的日志记录：
    // 1: <normal> 一条正常严重级别的消息
    // 2: <error> 一条错误严重级别的消息
    sink->set_formatter
    (
        expr::format("%1%: <%2%> %3%")
            % expr::attr< unsigned int >("LineID")
            % logging::trivial::severity
            % expr::smessage
    );

    logging::core::get()->add_sink(sink);
}
```

[完整代码](https://www.boost.org/doc/libs/1_86_0/libs/log/example/doc/tutorial_fmt_format.cpp)

`format`占位符接受格式字符串，其中对所有格式化参数的定位说明被指定。请注意，目前仅支持位置格式。相同的格式说明也可以与 `add_file_log` 和类似函数一起使用。

### Specialized formatters
该库提供了针对多种类型（例如日期、时间和命名范围）的专用格式化器。这些格式化器提供对格式化值的扩展控制。例如，可以使用与 [Boost.DateTime](https://www.boost.org/doc/libs/release/doc/html/date_time.html) 兼容的格式字符串描述日期和时间格式：

```cpp
void init()
{
    logging::add_file_log
    (
        keywords::file_name = "sample_%N.log",
        // 这将使汇写入看起来像这样的日志记录：
        // YYYY-MM-DD HH:MI:SS: <normal> 一条正常严重级别的消息
        // YYYY-MM-DD HH:MI:SS: <error> 一条错误严重级别的消息
        keywords::format =
        (
            expr::stream
                << expr::format_date_time< boost::posix_time::ptime >("TimeStamp", "%Y-%m-%d %H:%M:%S")
                << ": <" << logging::trivial::severity
                << "> " << expr::smessage
        )
    );
}
```

[完整代码](https://www.boost.org/doc/libs/1_86_0/libs/log/example/doc/tutorial_fmt_stream.cpp)

同样的格式化器也可以在 [Boost.Format](https://www.boost.org/doc/libs/release/libs/format/index.html) 风格的格式化器上下文中使用。

### String templates as formatters
在某些上下文中，文本模板被接受为格式化器。在这种情况下，库初始化支持代码会被调用以解析模板并重建适当的格式化器。在使用这种方法时需要注意一些注意事项，但这里足以简要描述模板格式。

```cpp
void init()
{
    logging::add_file_log
    (
        keywords::file_name = "sample_%N.log",
        keywords::format = "[%TimeStamp%]: %Message%"
    );
}
```

[完整代码](https://www.boost.org/doc/libs/1_86_0/libs/log/example/doc/tutorial_fmt_string.cpp)

在这里，格式参数接受这样的格式模板。模板可能包含多个用百分号（%）括起来的占位符。每个占位符必须包含一个属性值名称，以便在占位符中插入。`%Message%` 占位符将替换为日志记录消息。

---

⚠注意

文本格式模板不被汇后端的 `set_formatter` 方法接受。为了将文本模板解析为格式化器函数，必须调用 `parse_formatter` 函数。有关更多详细信息，请参阅这里。

---

### Custom formatting functions
您可以向支持格式化的汇后端添加自定义格式化器。格式化器实际上是一个函数对象，支持以下签名：

```cpp
void (logging::record_view const& rec, logging::basic_formatting_ostream< CharT >& strm);
```

这里的 `CharT` 是目标字符类型。每当日志记录视图 `rec` 通过过滤并要存储到日志中时，将调用格式化器。

---

❗提示

日志记录视图与记录非常相似。显著区别在于，视图是不可变的，并且实现了浅拷贝。格式化器和汇仅在记录视图上操作，这防止了它们在记录仍然可以被其他汇在其他线程中使用时修改记录。

---

格式化的记录应通过插入到标准库兼容的输出流 `strm` 中进行构成。以下是自定义格式化器函数用法的示例：

```cpp
void my_formatter(logging::record_view const& rec, logging::formatting_ostream& strm)
{
    // 获取 LineID 属性值并放入流中
    strm << logging::extract< unsigned int >("LineID", rec) << ": ";

    // 严重级别也是一样。
    // 如果使用属性关键字，则可能使用简化的语法。
    strm << "<" << rec[logging::trivial::severity] << "> ";

    // 最后，将记录消息放入流中
    strm << rec[expr::smessage];
}

void init()
{
    typedef sinks::synchronous_sink< sinks::text_ostream_backend > text_sink;
    boost::shared_ptr< text_sink > sink = boost::make_shared< text_sink >();

    sink->locked_backend()->add_stream(
        boost::make_shared< std::ofstream >("sample.log"));

    sink->set_formatter(&my_formatter);

    logging::core::get()->add_sink(sink);
}
```

[完整代码](https://www.boost.org/doc/libs/1_86_0/libs/log/example/doc/tutorial_fmt_custom.cpp)