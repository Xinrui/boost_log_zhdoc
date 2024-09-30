## Filtering revisited

我们之前已经涉及过过滤器，但只触及了表面。现在我们能够为日志记录添加属性并设置汇，我们可以构建任何所需的复杂过滤器。来看一个例子：

```cpp
BOOST_LOG_ATTRIBUTE_KEYWORD(line_id, "LineID", unsigned int)
BOOST_LOG_ATTRIBUTE_KEYWORD(severity, "Severity", severity_level)
BOOST_LOG_ATTRIBUTE_KEYWORD(tag_attr, "Tag", std::string)

void init()
{
    // 设置所有汇的通用格式化器
    logging::formatter fmt = expr::stream
        << std::setw(6) << std::setfill('0') << line_id << std::setfill(' ')
        << ": <" << severity << ">\t"
        << expr::if_(expr::has_attr(tag_attr))
           [
               expr::stream << "[" << tag_attr << "] "
           ]
        << expr::smessage;

    // 初始化汇
    typedef sinks::synchronous_sink< sinks::text_ostream_backend > text_sink;
    boost::shared_ptr< text_sink > sink = boost::make_shared< text_sink >();

    sink->locked_backend()->add_stream(
        boost::make_shared< std::ofstream >("full.log"));

    sink->set_formatter(fmt);

    logging::core::get()->add_sink(sink);

    sink = boost::make_shared< text_sink >();

    sink->locked_backend()->add_stream(
        boost::make_shared< std::ofstream >("important.log"));

    sink->set_formatter(fmt);

    sink->set_filter(severity >= warning || (expr::has_attr(tag_attr) && tag_attr == "IMPORTANT_MESSAGE"));

    logging::core::get()->add_sink(sink);

    // 添加属性
    logging::add_common_attributes();
}
```

[完整代码](https://www.boost.org/doc/libs/1_86_0/libs/log/example/doc/tutorial_filtering.cpp)

在这个示例中，我们初始化了两个汇——一个用于完整的日志文件，另一个只记录重要消息。两个汇都会使用相同的日志记录格式写入文本文件。首先，我们初始化格式化器并将其保存到 `fmt` 变量中。格式化器类型是一个类型擦除的函数对象，具有格式化器调用签名；它与 `boost::function` 或 `std::function` 类似，只不过它永远不会为空。过滤器也有类似的函数对象。

值得注意的是，格式化器本身包含了一个过滤器。正如您所看到的，格式中包含一个条件部分，该部分仅在日志记录包含 "Tag" 属性时才会出现。`has_attr` 谓词用于检查记录是否包含 "Tag" 属性值，并决定是否将其写入文件。我们使用属性关键字来指定谓词的属性名称和类型，但也可以在 `has_attr` 调用时指定它们。条件格式化器的更多细节可以在此处找到。

接下来是两个汇的初始化。第一个汇没有过滤器，这意味着它会将每条日志记录保存到文件中。我们对第二个汇调用 `set_filter`，以便只保存严重性不低于 `warning` 的日志记录或带有值为 "IMPORTANT_MESSAGE" 的 "Tag" 属性的日志记录。正如您所见，过滤器语法与普通 C++ 十分类似，特别是在使用属性关键字时。

与格式化器类似，过滤器也可以使用自定义函数。本质上，过滤器函数必须支持以下签名：

```cpp
bool (logging::attribute_value_set const& attrs);
```

当调用过滤器时，`attrs` 将包含一组完整的属性值，可用于决定是否应传递或抑制日志记录。如果过滤器返回 `true`，日志记录将被构建并由汇进一步处理；否则，记录将被丢弃。

[Boost.Phoenix](https://www.boost.org/doc/libs/release/libs/phoenix/doc/html/index.html) 在构建过滤器时非常有用。它允许自动从 `attrs` 集合中提取属性值，因为它的绑定实现与属性占位符兼容。之前的示例可以修改如下：

```cpp
bool my_filter(logging::value_ref< severity_level, tag::severity > const& level,
               logging::value_ref< std::string, tag::tag_attr > const& tag)
{
    return level >= warning || tag == "IMPORTANT_MESSAGE";
}

void init()
{
    // ...

    namespace phoenix = boost::phoenix;
    sink->set_filter(phoenix::bind(&my_filter, severity.or_none(), tag_attr.or_none()));

    // ...
}
```

如您所见，自定义过滤器接收作为独立参数的属性值，这些值被包装在 `value_ref` 模板中。此包装器包含一个可选引用，指向指定类型的属性值；如果日志记录包含所需类型的属性值，则该引用有效。`my_filter` 中使用的关系运算符可以无条件应用，因为它们会在引用无效时自动返回 `false`。其余部分由 `bind` 表达式处理，它会识别 `severity` 和 `tag_attr` 关键字，并在将其传递给 `my_filter` 之前提取相应的值。

---

📓注意

由于与 [`Boost.Phoenix`](https://www.boost.org/doc/libs/release/libs/phoenix/doc/html/index.html) 集成相关的限制（参见 [#7996](https://svn.boost.org/trac/boost/ticket/7996)），当与 `phoenix::bind` 或 `phoenix::function` 一起使用属性关键字时，必须明确指定属性值缺失时的回退策略。在上面的示例中，这是通过调用 `or_none` 完成的，如果找不到值，它会生成一个空的 `value_ref`。在其他上下文中，这种策略是默认的。还有其他策略可供选择。

---

您可以通过编译和运行[测试](https://www.boost.org/doc/libs/1_86_0/libs/log/example/doc/tutorial_filtering.cpp)来试验该功能。