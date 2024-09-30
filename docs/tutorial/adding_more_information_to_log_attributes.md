## Adding more information to log: Attributes

在前面的章节中，我们提到过属性及其属性值多次。这里我们将探讨如何利用属性为日志记录添加更多数据。

每个日志记录可以附加多个命名的属性值。属性可以表示关于日志记录发生时条件的任何重要信息，例如代码位置、可执行模块名称、当前日期和时间，或与特定应用程序和执行环境相关的任何数据。属性可以作为值生成器使用，这样每个参与的日志记录将返回不同的值。一旦属性生成了值，该值就与创建者无关，可以被过滤器、格式化器和汇使用。但要使用属性值，必须知道其名称和类型，或至少是一组可能的类型。库中实现了一些常用属性，您可以在文档中找到它们的值类型。

除了上述内容，如[设计概述部分](./../design_overview.md#design-overview)所述，属性有三种可能的作用域：特定源、特定线程和全局。当创建日志记录时，这三组属性的值会合并为一组，并传递给汇。这意味着属性的来源对汇没有区别。任何属性都可以在任何作用域中注册。注册时，属性会被赋予一个唯一的名称，以便进行查找。如果在多个作用域中发现同名属性，将优先考虑来自最具体作用域的属性，进行后续处理，包括过滤和格式化。这种行为允许使用在本地日志记录器中注册的属性来覆盖全局或线程作用域的属性，从而减少线程干扰。

接下来是属性注册过程的描述。

### Commonly used attributes

有一些属性几乎在所有应用程序中都会被使用。日志记录计数器和时间戳是很好的候选者。它们可以通过一次函数调用添加：

```cpp
logging::add_common_attributes();
```

此调用将全局注册属性 "LineID"、"TimeStamp"、"ProcessID" 和 "ThreadID"。"LineID" 属性是一个计数器，记录被创建时递增，第一条记录的标识符为 1。"TimeStamp" 属性始终返回当前时间（即创建日志记录的时间，而不是写入汇的时间）。最后两个属性标识每条日志记录所发出的进程和线程。

---

**📓注意**  

在单线程构建中，"ThreadID" 属性不会被注册。

--- 

**❗提示**  

默认情况下，当应用程序启动时，库中不会注册任何属性。应用程序必须在开始写入日志之前注册所有必要的属性。这可以作为库初始化的一部分完成。好奇的读者可能会想，简单日志记录是如何工作的。答案是，默认汇实际上不使用任何属性值（除了严重性级别）来组成输出。这是为了避免简单日志记录的任何初始化需求。一旦使用过滤器或格式化器及非默认汇，您就需要注册所需的属性。

---

`add_common_attributes` 函数是这里描述的几个便捷助手之一。

一些属性在日志记录器构造时会自动注册。例如，`severity_logger` 会注册一个特定源的属性 "Severity"，可用于为不同日志记录添加强调级别。例如：

```cpp
// 我们定义自己的严重性级别
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
    // 日志记录器在构造时隐式添加了源特定属性 'Severity'
    src::severity_logger< severity_level > slg;

    BOOST_LOG_SEV(slg, normal) << "一条普通消息";
    BOOST_LOG_SEV(slg, warning) << "发生了一些不好的事情，但我能应对";
    BOOST_LOG_SEV(slg, critical) << "一切崩溃了，快杀了我吧！";
}
```

---

**❗提示**

您可以通过为该类型定义 `operator <<` 来定义自己的严重性级别格式化规则。库的格式化器将自动使用该规则。

---

`BOOST_LOG_SEV` 宏与 `BOOST_LOG` 的功能类似，但它为日志记录器的 `open_record` 方法增加了一个额外的参数。`BOOST_LOG_SEV` 宏可以替换为以下等效的代码：

```cpp
void manual_logging()
{
    src::severity_logger< severity_level > slg;

    logging::record rec = slg.open_record(keywords::severity = normal);
    if (rec)
    {
        logging::record_ostream strm(rec);
        strm << "一条普通消息";
        strm.flush();
        slg.push_record(boost::move(rec));
    }
}
```

可以看到，`open_record` 可以接受命名参数。库提供的一些日志记录器类型支持这种附加参数，而用户在编写自己的日志记录器时也可以采用这种方法。

### More attributes

让我们看看我们在简单形式部分使用的 `add_common_attributes` 函数的内部实现。它可能看起来像这样：

```cpp
void add_common_attributes()
{
    boost::shared_ptr< logging::core > core = logging::core::get();
    core->add_global_attribute("LineID", attrs::counter< unsigned int >(1));
    core->add_global_attribute("TimeStamp", attrs::local_clock());

    // 其他属性略去以简化
}
```

这里的 `counter` 和 `local_clock` 组件是属性类，它们继承自通用属性接口。库提供了许多其他属性类，包括在值获取时调用某个函数对象的 `function` 属性。例如，我们可以以类似的方式注册一个 `named_scope` 属性：

```cpp
core->add_global_attribute("Scope", attrs::named_scope());
```

这将使我们能够在每个日志记录中存储作用域名称。以下是其用法示例：

```cpp
void named_scope_logging()
{
    BOOST_LOG_NAMED_SCOPE("named_scope_logging");

    src::severity_logger< severity_level > slg;

    BOOST_LOG_SEV(slg, normal) << "来自函数 named_scope_logging 的问候！";
}
```

特定于日志记录器的属性同样有用。严重性级别和通道名称是最明显的候选者，可以在源级别实现。您也可以向日志记录器添加更多属性，如下所示：

```cpp
void tagged_logging()
{
    src::severity_logger< severity_level > slg;
    slg.add_attribute("Tag", attrs::constant< std::string >("我的标签值"));

    BOOST_LOG_SEV(slg, normal) << "这是带标签的记录";
}
```

现在，通过此日志记录器生成的所有日志记录都将附带特定属性。该属性值可以稍后用于过滤和格式化。

属性的另一个良好用途是标记由应用程序不同部分生成的日志记录，以突出与单个进程相关的活动。可以实现一个粗略的性能分析工具，以检测性能瓶颈。例如：

```cpp
void timed_logging()
{
    BOOST_LOG_SCOPED_THREAD_ATTR("Timeline", attrs::timer());

    src::severity_logger< severity_level > slg;
    BOOST_LOG_SEV(slg, normal) << "开始计时嵌套函数";

    logging_function();

    BOOST_LOG_SEV(slg, normal) << "停止计时嵌套函数";
}
```

现在，从 `logging_function` 函数生成的每条日志记录，或它调用的任何其他函数，都会包含 "Timeline" 属性，记录自属性注册以来经过的高精度时间段。根据这些读数，可以检测出代码的哪些部分需要更多或更少的执行时间。"Timeline" 属性将在退出 `timed_logging` 函数的作用域时注销。

有关库提供的属性的详细描述，请参见属性部分。此部分的完整代码可在此处找到。

### Defining attribute placeholders

正如我们在接下来的章节中将看到的，定义一个描述应用程序使用的特定属性的关键字是很有用的。这个关键字可以参与过滤和格式化表达式，就像我们在前面的章节中使用的严重性占位符。例如，定义之前示例中使用的一些属性的占位符可以这样写：

```cpp
BOOST_LOG_ATTRIBUTE_KEYWORD(line_id, "LineID", unsigned int)
BOOST_LOG_ATTRIBUTE_KEYWORD(severity, "Severity", severity_level)
BOOST_LOG_ATTRIBUTE_KEYWORD(tag_attr, "Tag", std::string)
BOOST_LOG_ATTRIBUTE_KEYWORD(scope, "Scope", attrs::named_scope::value_type)
BOOST_LOG_ATTRIBUTE_KEYWORD(timeline, "Timeline", attrs::timer::value_type)
```

每个宏定义了一个关键字。第一个参数是占位符名称，第二个是属性名称，最后一个参数是属性值类型。定义后，可以在模板表达式和库的其他上下文中使用占位符。有关定义属性关键字的更多详细信息，请参见[此处](https://www.boost.org/doc/libs/1_86_0/libs/log/doc/html/log/detailed/expressions.html#log.detailed.expressions.attr_keywords)。
