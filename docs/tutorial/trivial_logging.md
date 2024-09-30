## Trivial logging

对于那些不想阅读大量晦涩手册，只需要一个简单日志工具的用户，这里有一个例子：

```cpp
#include <boost/log/trivial.hpp>

int main(int, char*[])
{
    BOOST_LOG_TRIVIAL(trace) << "A trace severity message";
    BOOST_LOG_TRIVIAL(debug) << "A debug severity message";
    BOOST_LOG_TRIVIAL(info) << "An informational severity message";
    BOOST_LOG_TRIVIAL(warning) << "A warning severity message";
    BOOST_LOG_TRIVIAL(error) << "An error severity message";
    BOOST_LOG_TRIVIAL(fatal) << "A fatal severity message";

    return 0;
}
```

[完整代码](https://www.boost.org/doc/libs/1_86_0/libs/log/example/doc/tutorial_trivial.cpp)

`BOOST_LOG_TRIVIAL` 宏接受一个严重性级别，并生成一个支持插入运算符的流式对象。通过这段代码，日志消息将打印在控制台上。正如你所看到的，这种库的使用模式与 `std::cout` 非常相似。然而，这个库提供了一些优势：

1. 除了记录消息外，输出中的每条日志记录还包含时间戳、当前线程标识符和严重性级别。
2. 可以安全地从不同线程同时写入日志，日志消息不会被破坏。
3. 如后面将展示的，可以应用过滤。

需要说明的是，这个宏以及库中提供的其他类似宏并不是唯一的接口。正如后面将展示的，可以在不使用任何宏的情况下发出日志记录。
