## Trivial logging with filters

虽然严重性级别可以用于信息性目的，但通常你会希望应用过滤器，仅输出重要记录并忽略其余内容。可以通过在库核心中设置全局过滤器来轻松实现，如下所示：

```cpp
void init()
{
    logging::core::get()->set_filter
    (
        logging::trivial::severity >= logging::trivial::info
    );
}

int main(int, char*[])
{
    init();

    BOOST_LOG_TRIVIAL(trace) << "A trace severity message";
    BOOST_LOG_TRIVIAL(debug) << "A debug severity message";
    BOOST_LOG_TRIVIAL(info) << "An informational severity message";
    BOOST_LOG_TRIVIAL(warning) << "A warning severity message";
    BOOST_LOG_TRIVIAL(error) << "An error severity message";
    BOOST_LOG_TRIVIAL(fatal) << "A fatal severity message";

    return 0;
}
```

[完整代码](https://www.boost.org/doc/libs/1_86_0/libs/log/example/doc/tutorial_trivial_flt.cpp)

现在，如果我们运行这个代码示例，前两个日志记录将被忽略，而剩下的四个将显示在控制台上。

--- 

**⚠重要提示**

请记住，只有当记录通过过滤器时，流表达式才会被执行。不要在流表达式中指定对业务至关重要的调用，因为如果记录被过滤掉，这些调用可能不会被执行。

---

关于过滤器设置表达式，需要说几句话。由于我们正在设置全局过滤器，因此必须获取[日志核心](./../detailed_features_description/core_facilities.md#logging-core)实例。这就是 `logging::core::get()` 的作用——它返回指向核心单例的指针。日志核心的 `set_filter` 方法用于设置全局过滤函数。

在这个例子中，过滤器是作为 [Boost.Phoenix](https://www.boost.org/doc/libs/release/libs/phoenix/doc/html/index.html) 的 lambda 表达式构建的。在我们的例子中，这个表达式由一个逻辑谓词组成，其左侧参数是描述要检查的属性的占位符，右侧参数是要对比的值。`severity` 关键字是库提供的占位符，它在模板表达式中标识严重性属性值；该值的名称应为 "Severity"，类型为 `severity_level`。在简单日志记录的情况下，该属性由库自动提供，用户只需在日志语句中提供其值。占位符和排序运算符结合起来创建了一个函数对象，日志核心将调用该对象来过滤日志记录。结果，只有严重性级别不低于 info 的日志记录会通过过滤并显示在控制台上。

可以构建更复杂的过滤器，将这样的逻辑谓词组合在一起，甚至可以定义自己的函数（包括 C++11 lambda 函数）作为过滤器。我们将在接下来的部分中再次讨论过滤。
