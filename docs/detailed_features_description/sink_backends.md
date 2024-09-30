## Sink backends

### Text stream backend
```cpp
#include <boost/log/sinks/text_ostream_backend.hpp>
```

文本输出流汇后端是库中提供的最通用的后端。该后端在 `basic_text_ostream_backend` 类模板中实现（`text_ostream_backend` 和 `wtext_ostream_backend` 提供了狭义和宽字符支持的便捷类型别名）。它支持将日志记录格式化为字符串并放入一个或多个流中。每个附加的流都会得到相同的格式化结果，因此，如果你需要为不同的流以不同的方式格式化日志记录，则需要创建多个汇——每个汇都有自己的格式化程序。

该后端还提供了一个可能在调试应用程序时非常有用的功能。通过 `auto_flush` 方法，可以告诉汇在每次写入日志记录后自动刷新所有附加流的缓冲区。这当然会降低日志记录性能，但在应用程序崩溃的情况下，有很大机会不会丢失最后的日志记录。

```cpp
void init_logging()
{
    boost::shared_ptr< logging::core > core = logging::core::get();

    // 创建后端并附加几个流
    boost::shared_ptr< sinks::text_ostream_backend > backend =
        boost::make_shared< sinks::text_ostream_backend >();
    backend->add_stream(
        boost::shared_ptr< std::ostream >(&std::clog, boost::null_deleter()));
    backend->add_stream(
        boost::shared_ptr< std::ostream >(new std::ofstream("sample.log")));

    // 在每次写入日志记录后启用自动刷新
    backend->auto_flush(true);

    // 将其包装到前端并在核心中注册。
    // 后端需要在前端中进行同步。
    typedef sinks::synchronous_sink< sinks::text_ostream_backend > sink_t;
    boost::shared_ptr< sink_t > sink(new sink_t(backend));
    core->add_sink(sink);
}
```

### Text file backend

```cpp
#include <boost/log/sinks/text_file_backend.hpp>
```

虽然可以使用文本流后端将日志写入文件，但该库还提供了一个特殊的接收后端，具有适合文件日志记录的扩展功能。功能包括：

- 基于文件大小和/或时间的日志文件轮换
- 灵活的日志文件命名
- 将轮换的文件放置在文件系统中的特殊位置
- 删除最旧的文件以释放文件系统上的更多空间
- 与文本流后端类似，文件接收后端还支持自动刷新功能

该后端称为 `text_file_backend`。

---

❗警告

此接收器内部使用 [Boost.Filesystem](https://www.boost.org/doc/libs/release/libs/filesystem/doc/index.htm)，这可能会导致进程终止时出现问题。有关更多详细信息，请参见此处。

---

#### File rotation
```cpp
#include <boost/log/sinks/text_file_backend.hpp>
```

文件轮换发生在接收器检测到一个或多个轮换条件满足并需要创建新文件时。它由接收后端实现，包括以下步骤：

1. 如果日志文件当前处于打开状态，调用文件关闭处理程序并关闭该文件。
2. 如果设置了目标文件名模式，则生成新的目标文件名并将日志文件重命名为生成的名称。
3. 如果配置了文件收集器，则将日志文件传递给它进行收集。此时，文件收集器可以删除旧的日志文件以释放空间，并将新文件移动到目标存储。
4. 当需要写入新日志记录时，为新文件生成新文件名并创建该名称的文件。如果启用了文件追加，并且该名称的文件已存在，则文件将以追加模式打开，而不是覆盖。

重要的是要注意，这个过程中涉及三种文件名或路径：

- 用于创建或打开日志文件以进行主动写入的文件名。这被称为活动文件名，由接收后端构造函数的 `file_name` 命名参数指定，或通过调用 `set_file_name_pattern` 方法来指定。
- 在日志文件关闭并即将被收集时生成的文件名。这被称为目标文件名，因为它定义了日志文件在目标存储中的命名方式。目标文件名是可选的，可以通过 `target_file_name` 命名参数指定，或通过调用 `set_target_file_name_pattern` 方法来指定。
- 目标存储位置，即由文件收集器管理的包含以前轮换日志文件的目录。多个接收器可以共享同一个目标存储。

文件名模式和轮换条件可以在构造 `text_file_backend` 后端时指定。

```cpp
void init_logging()
{
    boost::shared_ptr< logging::core > core = logging::core::get();

    boost::shared_ptr< sinks::text_file_backend > backend =
        boost::make_shared< sinks::text_file_backend >(
            keywords::file_name = "file.log",                                              // 活动文件名模式
            keywords::target_file_name = "file_%5N.log",                                   // 目标文件名模式
            keywords::rotation_size = 5 * 1024 * 1024,                                     // 达到 5 MiB 大小时轮换文件…
            keywords::time_based_rotation = sinks::file::rotation_at_time_point(12, 0, 0)  // ……或每天中午轮换，以先到者为准
        );

    // 包装到前端并注册到核心。
    // 后端需要在前端进行同步。
    typedef sinks::synchronous_sink< sinks::text_file_backend > sink_t;
    boost::shared_ptr< sink_t > sink(new sink_t(backend));

    core->add_sink(sink);
}
```

---

📓注意

文件大小在轮换时可能不精确。实现计算写入文件的字节数，但底层 API 可能会引入额外的辅助数据，这会增加日志文件在磁盘上的实际大小。例如，众所周知，Windows 和 DOS 操作系统对换行符有特殊处理。每个换行符以两个字节的序列 0x0D 0x0A 而不是单个 0x0A 写入。其他平台特定的字符转换也已知。压缩文件系统上，实际的磁盘大小也可能小于写入的字符数量。

---

基于时间的轮换不仅限于时间点。可用的选项有：

1. **时间点轮换**：`rotation_at_time_point` 类。此类轮换在达到指定时间点时发生。以下变体可用：


- 每天轮换，在指定时间。这就是上面代码片段中展示的内容：
  
```cpp
sinks::file::rotation_at_time_point(12, 0, 0)
```

- 在每周的指定天轮换，在指定时间。例如，这将使文件轮换发生在每周二的午夜：
  
```cpp
sinks::file::rotation_at_time_poin(date_time::Tuesday, 0, 0, 0)
```

如果是午夜，则可以省略时间：

```cpp
sinks::file::rotation_at_time_poin(date_time::Tuesday)
```
- 在每月的指定天轮换，在指定时间。例如，以下代码将每月的第一天轮换文件：

```cpp
sinks::file::rotation_at_time_poin(gregorian::greg_day(1), 0, 0, 0)
```

与星期几类似，午夜是隐含的：

```cpp
sinks::file::rotation_at_time_poin(gregorian::greg_day(1))
```

2. **时间间隔轮换**：`rotation_at_time_interval` 类。使用此谓词，轮换不受任何时间点的限制，而是在指定时间间隔自上次轮换后经过时发生。以下代码每小时轮换：
```cpp
sinks::file::rotation_at_time_interva(posix_time::hours(1))
```

如果以上都不适用，则可以为基于时间的轮换指定自己的谓词。该谓词应不带参数并返回布尔值（返回 true 表示轮换应发生）。例如：
```cpp
bool is_it_time_to_rotate();
```

```cpp
void init_logging()
{
    // ...

    boost::shared_ptr< sinks::text_file_backend > backend =
        boost::make_shared< sinks::text_file_backend >(
            keywords::file_name = "file.log",
            keywords::target_file_name = "file_%5N.log",
            keywords::time_based_rotation = &is_it_time_to_rotate
        );

    // ...
}
```

> **注意**  
> 日志文件轮换发生在尝试向文件写入新日志记录时。因此，基于时间的轮换也不是严格的阈值。一旦库检测到应发生轮换，就会发生轮换。

除了基于时间和大小的文件轮换外，后端在销毁时也默认执行轮换。这是为了在程序终止后维护目标目录中收集的所有日志文件，并确保临时日志文件不会堆积在接收器后端写入的目录中。可以通过后端构造函数的 `enable_final_rotation` 参数或后端的类似命名方法禁用此行为：

```cpp
void init_logging()
{
    // ...

    boost::shared_ptr< sinks::text_file_backend > backend =
        boost::make_shared< sinks::text_file_backend >(
            keywords::file_name = "file.log",
            keywords::target_file_name = "file_%5N.log",
            keywords::enable_final_rotation = false
        );

    // ...
}
```

活动文件名和目标文件名模式都可以包含多个通配符，例如上面的示例中所示。支持的占位符包括：

- 当前日期和时间组件。占位符符合 Boost.DateTime 库中指定的格式。
- 文件计数器（%N）以及可选的宽度规格（类似于 printf 格式）。文件计数器将始终为十进制，填充到指定宽度。
- 百分号（%%）。

以下是一些快速示例：

| 模板                     | 扩展为                          |
|-------------------------|-------------------------------|
| `file_%N.log`          | `file_1.log`, `file_2.log`... |
| `file_%3N.log`         | `file_001.log`, `file_002.log`... |
| `file_%Y%m%d.log`      | `file_20080705.log`, `file_20080706.log`... |
| `file_%Y-%m-%d_%H-%M-%S.%N.log` | `file_2008-07-05_13-44-23.1.log`, `file_2008-07-06_16-00-10.2.log`... |

> **重要**  
> 尽管所有 Boost.DateTime 格式说明符都有效，但如果您打算扫描旧日志文件，则对某些格式有限制。有关此功能的讨论，请参见此处。

请注意，如上所述，活动和目标文件名在不同时间生成。具体而言，活动文件名在日志文件首次创建时生成，而目标文件名在文件关闭时生成。用于构造这些文件名的时间戳将反映这一差异。

> **提示**  
> 当需要文件追加时，建议避免在活动文件名模式中使用任何占位符。否则，由于活动日志文件名不同，将不会发生追加。可以使用目标文件名模式在轮换后向日志文件添加时间戳或计数器。
> 
#### File open and close handlers
#### Managing rotated files
#### Scanning for rotated files
#### Appending to the previously written files

### Text multi-file backend
### Text IPC message queue backend
### Syslog backend
### Windows debugger output backend
### Windows event log backends

