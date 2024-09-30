## Core facilities

### Logging records
```cpp  
#include <boost/log/core/record.hpp>
```

所有日志库处理的信息都打包在一个类型为 `record` 的对象中。所有附加的数据，包括消息文本，都表示为命名的属性值，过滤器、格式化器和汇可以获取和处理这些属性值。可以通过不同的方式访问特定的属性值，以下是几个快速示例：

- 通过值的访问和提取。

```cpp  
enum severity_level { ... };
std::ostream& operator<< (std::ostream& strm, severity_level level);

struct print_visitor
{
    typedef void result_type;
    result_type operator() (severity_level level) const
    {
        std::cout << level << std::endl;
    };
};

// 通过visitation API打印严重级别
void print_severity_visitation(logging::record const& rec)
{
    logging::visit<severity_level>("Severity", rec, print_visitor());
}

// 通过extraction API打印严重级别
void print_severity_extraction(logging::record const& rec)
{
    logging::value_ref<severity_level> level = logging::extract<severity_level>("Severity", rec);
    std::cout << level << std::endl;
}
```

- 通过搜索 `record` 的 `attribute_values` 方法可以访问的属性值集合。

```cpp  
// 通过搜索属性值打印严重级别
void print_severity_lookup(logging::record const& rec)
{
    logging::attribute_value_set const& values = rec.attribute_values();
    logging::attribute_value_set::const_iterator it = values.find("Severity");
    if (it != values.end())
    {
        logging::attribute_value const& value = it->second;

        // 单个属性值也可以访问或提取
        std::cout << value.extract<severity_level>() << std::endl;
    }
}
```

- 通过使用属性关键字的下标操作符。这实际上是值提取API的便利包装。

```cpp  
BOOST_LOG_ATTRIBUTE_KEYWORD(severity, "Severity", severity_level)

// 通过下标操作符打印严重级别
void print_severity_subscript(logging::record const& rec)
{
    // 使用属性关键字传达值的名称和类型
    logging::value_ref<severity_level, tag::severity> level = rec[severity];
    std::cout << level << std::endl;
}
```

日志记录不能复制，只能移动。`record` 可以默认构造，在这种情况下，它处于空状态；此类记录大多不可用，不应传递给库进行处理。非空的日志记录只能由[日志核心](#logging-core)在成功过滤后创建。非空日志记录包含从属性获取的属性值。更多属性值可以在过滤后添加到非空记录中，添加的值不会影响过滤结果，但可以用于格式化器和汇。

在多线程环境中，构造完成后，非空日志记录被认为与当前线程绑定，因为它可能引用一些线程特定的资源。例如，记录可能包含引用存储在堆栈中的命名范围列表的属性值。因此，日志记录不能在不同线程之间传递。

#### Record views
```cpp  
#include <boost/log/core/record_view.hpp>
```

虽然 `record` 用于填充信息，但库使用另一种类型来实际处理它。记录视图提供与记录类似的接口，但有几个显著的区别：

- 记录视图是不可变的。这防止了格式化器和汇在处理记录时修改它。
- 记录视图是可复制的。由于其内容是常量，复制操作是浅层的，因此代价较低。

库将通过调用 `lock` 方法自动从记录中创建记录视图。该调用还确保如果汇是异步的，则生成的视图不会与当前线程相关联。`lock` 调用是一次性操作，之后记录将处于空状态。用于与属性值交互的所有API也适用于记录视图，可以在自定义格式化器和汇中使用。

---

### Logging core
```cpp  
#include <boost/log/core/core.hpp>
```

日志核心是一个中央枢纽，提供以下功能：

- 维护全局和线程特定的属性集。
- 执行日志记录的全局过滤。
- 通过应用特定于汇的过滤器在汇之间调度日志记录。
- 提供异常处理程序的全局钩子。
- 为日志源提供输入点以写入日志记录。
- 提供可以用于强制所有日志汇同步状态的刷新方法。

日志核心是一个全局单例，因此每个日志源都可以访问它。核心实例可以通过静态方法 `get` 访问。

```cpp
void foo()
{
    boost::shared_ptr< logging::core > core = logging::core::get();

    // ...
}
```

#### Attribute sets
为了向核心添加或移除全局或线程特定的属性，有相应的方法：`add_global_attribute`、`remove_global_attribute`、`add_thread_attribute` 和 `remove_thread_attribute`。属性集提供了类似于 `std::map` 的接口，因此 `add_*` 方法接受一个属性名称字符串（键）和一个指向属性的指针（映射值），并返回一个迭代器和布尔值的对，就像 `std::map< ... >::insert` 所做的那样。`remove_*` 方法接受指向之前添加的属性的迭代器。

```cpp
void foo()
{
    boost::shared_ptr< logging::core > core = logging::core::get();

    // 添加全局属性
    std::pair< logging::attribute_set::iterator, bool > res =
        core->add_global_attribute("LineID", attrs::counter< unsigned int >());

    // ...

    // 移除添加的属性
    core->remove_global_attribute(res.first);
}
```

---

提示

需要指出的是，日志核心的所有方法在多线程环境中都是线程安全的。然而，这对其他组件，如迭代器或属性集，可能不成立。

---

可以获取整个属性集（全局或线程特定）的副本或将其安装到核心中。方法 `get_global_attributes`、`set_global_attributes`、`get_thread_attributes` 和 `set_thread_attributes` 用于此目的。

---

⚠警告

在将整个属性集安装到核心后，之前由相应的 `add_*` 方法返回的所有迭代器都将失效。特别是，这会影响作用域属性，因此用户在切换属性集时必须小心。

---

#### Global filtering
全局过滤由过滤函数对象处理，可以通过 `set_filter` 方法提供。有关创建过滤器的更多信息将在本节中出现。在这里，仅需说明过滤器接受一组属性值并返回一个布尔值，以指示具有这些属性值的日志记录是否通过了过滤。全局过滤应用于应用程序中的每个日志记录，因此可以快速消除过多的日志记录。

全局过滤器可以通过 `reset_filter` 方法移除。当核心中未设置过滤器时，假设没有记录被过滤掉。这是日志核心初始构造后的默认状态。

```cpp
enum severity_level
{
    normal,
    warning,
    error,
    critical
};

void foo()
{
    boost::shared_ptr< logging::core > core = logging::core::get();

    // 设置全局过滤器，只记录错误消息
    core->set_filter(expr::attr< severity_level >("Severity") >= error);

    // ...
}
```

核心还提供了另一种禁用日志记录的方法。通过调用 `set_logging_enabled` 并传入布尔参数，可以完全禁用或重新启用日志记录，包括应用过滤。使用此方法禁用日志记录在应用性能方面可能比设置一个总是失败的全局过滤器更有益。

#### Sink management
在应用全局过滤后，日志汇开始工作。为了添加和移除汇，核心提供了 `add_sink` 和 `remove_sink` 方法。这两个方法都接受指向汇的指针。`add_sink` 将汇添加到核心中，如果尚未添加的话。`remove_sink` 方法将在内部的已添加汇列表中查找提供的汇，并在找到时将其移除。核心内部处理汇的顺序是未指定的。

```cpp
void foo()
{
    boost::shared_ptr< logging::core > core = logging::core::get();

    // 设置一个将日志记录写入控制台的汇
    boost::shared_ptr< sinks::text_ostream_backend > backend =
        boost::make_shared< sinks::text_ostream_backend >();
    backend->add_stream(
        boost::shared_ptr< std::ostream >(&std::clog, boost::null_deleter()));

    typedef sinks::unlocked_sink< sinks::text_ostream_backend > sink_t;
    boost::shared_ptr< sink_t > sink = boost::make_shared< sink_t >(backend);
    core->add_sink(sink);

    // ...

    // 移除汇
    core->remove_sink(sink);
}
```

你可以在以下部分阅读有关汇设计的更多内容：汇前端和汇后端。

#### Exception handling
核心提供了一种设置集中式异常处理的方法。如果在过滤或处理添加的汇中的某个汇时发生异常，核心将调用已通过 `set_exception_handler` 方法安装的异常处理程序。异常处理程序是一个无参函数对象，它在 `catch` 子句内被调用。该库提供工具以简化异常处理程序的构造。

---

❗提示

日志核心中的异常处理程序是全局的，因此旨在对错误执行一些通用操作。日志汇和源也提供异常处理功能（请参见此处和此处），可以用于进行更精细的错误处理。

---

```cpp
struct my_handler
{
    typedef void result_type;

    void operator() (std::runtime_error const& e) const
    {
        std::cout << "std::runtime_error: " << e.what() << std::endl;
    }
    void operator() (std::logic_error const& e) const
    {
        std::cout << "std::logic_error: " << e.what() << std::endl;
        throw;
    }
};

void init_exception_handler()
{
    // 设置一个全局异常处理程序，将为指定的异常类型调用 my_handler::operator()
    logging::core::get()->set_exception_handler(logging::make_exception_handler<
        std::runtime_error,
        std::logic_error
    >(my_handler()));
}
```

#### Feeding log records
日志核心的一个重要功能是为所有日志源提供一个输入点，以便将日志记录输入。这是通过 `open_record` 和 `push_record` 方法实现的。

第一个方法用于启动记录日志的过程。它接受特定于源的属性集。该方法构建了全局、线程特定和源特定的三组属性的共同属性值，并应用过滤。如果过滤成功，即至少一个汇接受具有这些属性值的记录，则该方法返回一个非空的记录对象，可以用于填写日志记录消息。如果过滤失败，则返回一个空的记录对象。

当日志源准备完成日志记录过程时，必须调用 `push_record` 方法，并传入由 `open_record` 方法返回的记录。请注意，不应使用空记录调用 `push_record`。记录应作为右值引用传递。在调用期间，记录视图将从记录构造。然后，该视图将传递给在过滤过程中接受它的汇。这可能涉及记录格式化和进一步处理，例如存储到文件或通过网络发送。之后，可以销毁记录对象。

```cpp
void logging_function(logging::attribute_set const& attrs)
{
    boost::shared_ptr< logging::core > core = logging::core::get();

    // 尝试打开一个日志记录
    logging::record rec = core->open_record(attrs);
    if (rec)
    {
        // 好的，记录已被接受。现在构建消息。
        logging::record_ostream strm(rec);
        strm << "Hello, World!";
        strm.flush();

        // 将记录传递给汇。
        core->push_record(boost::move(rec));
    }
}
```

所有这些逻辑通常在库提供的记录器和宏中被隐藏。然而，对于开发新日志源的人来说，这可能是有用的。