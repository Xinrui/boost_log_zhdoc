## Wide character logging

该库支持记录包含国家字符的字符串。基本上有两种方式来实现这一点。在类似 UNIX 的系统中，通常使用某种多字节字符编码（例如 UTF-8）来表示国家字符。在这种情况下，可以像记录普通 ASCII 日志一样使用该库，不需要额外的设置。

在 Windows 系统中，常见的做法是使用宽字符串来表示国家字符。此外，大多数系统 API 都是面向宽字符的，这要求 Windows 特定的汇也支持宽字符串。另一方面，像文本文件汇这样的通用汇是面向字节的（因为，毕竟，文件中存储的是字节，而不是字符）。这就要求库在需要时对字符编码进行转换。为了设置这个功能，必须给汇指定一个带有适当 `codecvt` 的 `locale`。可以使用 [Boost.Locale](https://www.boost.org/doc/libs/release/libs/locale/doc/html/index.html) 来生成这样的 `locale`。我们来看一个示例：

```cpp
// 声明属性关键字
BOOST_LOG_ATTRIBUTE_KEYWORD(severity, "Severity", severity_level)
BOOST_LOG_ATTRIBUTE_KEYWORD(timestamp, "TimeStamp", boost::posix_time::ptime)

void init_logging()
{
    boost::shared_ptr< sinks::synchronous_sink< sinks::text_file_backend > > sink = logging::add_file_log
    (
        "sample.log",
        keywords::format = expr::stream
            << expr::format_date_time(timestamp, "%Y-%m-%d, %H:%M:%S.%f")
            << " <" << severity.or_default(normal)
            << "> " << expr::message
    );

    // 根据 imbue() 设置的 locale，汇将按需要进行字符编码转换
    std::locale loc = boost::locale::generator()("en_US.UTF-8");
    sink->imbue(loc);

    // 添加一些常用属性，例如时间戳和记录计数器
    logging::add_common_attributes();
}
```

首先，让我们看一下传递给 `format` 参数的格式化器。我们使用一个窄字符格式化器来初始化汇，因为文本文件汇处理字节。格式化器可以使用宽字符串，但不能在格式化字符串中使用，例如 `format_date_time` 函数中使用的字符串。另外，请注意，我们使用了 `message` 关键字来表示日志记录消息。这个占位符同时支持窄字符和宽字符消息，因此格式化器可以处理两者。作为格式化过程的一部分，库将使用设置的 `locale` 将宽字符消息转换为多字节编码，而我们将其设置为 UTF-8。

---

❗提示

属性值也可以包含宽字符串。像日志记录消息一样，这些字符串会根据设置的 `locale` 被转换为目标字符编码。

---

这里缺少的一个部分是我们的 `severity_level` 类型定义。该类型只是一个枚举，但如果我们希望它的格式化同时支持窄字符和宽字符汇，那么它的流操作符必须是一个模板。如果我们创建了多个具有不同字符类型的汇，这将非常有用。

```cpp
enum severity_level
{
    normal,
    notification,
    warning,
    error,
    critical
};

template< typename CharT, typename TraitsT >
inline std::basic_ostream< CharT, TraitsT >& operator<< (
    std::basic_ostream< CharT, TraitsT >& strm, severity_level lvl)
{
    static const char* const str[] =
    {
        "normal",
        "notification",
        "warning",
        "error",
        "critical"
    };
    if (static_cast< std::size_t >(lvl) < (sizeof(str) / sizeof(*str)))
        strm << str[lvl];
    else
        strm << static_cast< int >(lvl);
    return strm;
}
```

现在我们可以输出日志记录了。我们可以使用带有 `w` 前缀的日志记录器来构建宽字符消息。

```cpp
void test_narrow_char_logging()
{
    // 窄字符日志记录仍然有效
    src::logger lg;
    BOOST_LOG(lg) << "Hello, World! This is a narrow character message.";
}

void test_wide_char_logging()
{
    src::wlogger lg;
    BOOST_LOG(lg) << L"Hello, World! This is a wide character message.";

    // 也支持国家字符
    const wchar_t national_chars[] = { 0x041f, 0x0440, 0x0438, 0x0432, 0x0435, 0x0442, L',', L' ', 0x043c, 0x0438, 0x0440, L'!', 0 };
    BOOST_LOG(lg) << national_chars;

    // 现在，尝试使用严重性记录日志
    src::wseverity_logger< severity_level > slg;
    BOOST_LOG_SEV(slg, normal) << L"A normal severity message, will not pass to the file";
    BOOST_LOG_SEV(slg, warning) << L"A warning severity message, will pass to the file";
    BOOST_LOG_SEV(slg, error) << L"An error severity message, will pass to the file";
}
```

如您所见，宽字符消息的构建与窄字符日志记录类似。请注意，您可以同时使用窄字符和宽字符日志记录；所有记录都将由我们的文件汇处理。此示例的完整代码可以在[此处](https://www.boost.org/doc/libs/1_86_0/libs/log/example/wide_char/main.cpp)找到。

需要注意的是，一些汇（主要是 Windows 特定的汇）允许指定目标字符类型。当日志记录中预期包含国家字符时，应该始终使用 `wchar_t` 作为目标字符类型，因为汇将使用宽字符操作系统 API 来处理日志记录。在这种情况下，所有窄字符字符串将在格式化时使用设置的 `locale` 进行宽化。