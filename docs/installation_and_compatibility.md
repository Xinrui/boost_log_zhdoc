# Installation and compatibility

## Supported compilers and platforms
该库应能够在合理合规的 C++11 编译器上构建和运行。该库已在以下平台上成功构建和测试：

- **Windows 10**：MSVC 14.0 及更新版本；MinGW32 与 gcc 5.x；MinGW-w64 与 gcc 6.x 及更新版本。
- **Cygwin 和 Cygwin64**：gcc 7.x 及更新版本。
- **Linux**：GCC 5.x 及更新版本。
- **Linux**：Clang 3.5 及更新版本。

以下编译器/平台不受支持，可能无法编译该库：

- 不支持 C++11 的编译器。
- 带有非 C++11 标准库的 C++11 编译器（如使用 GCC 4.2 的 Clang 与 libstdc++）。请使用 C++11 模式下的 C++11 标准库。
- MSVC 12.0 及更早版本。
- GCC 4.4 及更早版本。
- Borland C++ 5.5.1（免费版本）。更新版本可能有效，也可能无效。
- Solaris Studio 12.3 及更早版本。
- Windows 9x、ME、NT4、2000 及更早版本不受支持。

Boost.Log 应与 Boost 支持的所有硬件架构兼容。然而，在 32 位 x86 架构下，库至少需要 i586 类 CPU 来运行。

**针对 GCC 用户的说明**

自 4.5 版本以来，GCC 支持链接时优化（LTO），此时大多数优化和二进制代码生成发生在链接阶段。这允许执行更高级的优化并生成更快的代码。不幸的是，它与包含需要使用不同编译器选项编译的源文件的项目不兼容。Boost.Log 就是这样一个项目，其中某些部分包含针对现代 CPU 的优化，无法在旧 CPU 上运行。启用 Boost.Log 的 LTO 会生成与旧 CPU 不兼容的二进制文件（GCC 错误 61043、77845），可能导致运行时崩溃。因此，Boost.Log 不支持 LTO。

使用 `-march=native` 命令行参数时，可能会出现导致编译失败的 GCC 错误。建议避免使用 `-march=native` 参数（或 `instruction-set=native` b2 属性），而是明确指定目标 CPU（例如 `instruction-set=sandy-bridge`）。

**针对 MinGW、Cygwin 和 Visual Studio Express Edition 用户的说明**

为了使用这些编译器编译该库，需要进行特殊准备。首先，对于 MinGW 或 Cygwin，请确保已安装最新版本的 GCC。使用 GCC 3.x 可能会导致编译失败。

其次，该库在某些情况下需要一个消息编译器工具（mc.exe），该工具在 MinGW、Cygwin 和某些版本的 MSVC Express Edition 中不可用。通常，库的构建脚本会自动检测系统上是否存在消息编译器，并在未找到时禁用与事件日志相关的库部分。如果需要事件日志支持但未在系统上找到，可以通过三种方式解决此问题。推荐的解决方案是获取原始的 mc.exe。该工具可在 Windows SDK 中找到，可以从 Microsoft 网站免费下载（例如，在此处）。此外，该工具应在 Visual Studio 2010 Express Edition 中可用。在编译过程中，mc.exe 应该在您的 PATH 环境变量的某个目录中可访问。

另一种方法是尝试使用 MinGW 和 Cygwin 分发的 windmc.exe 工具，该工具是原始 mc.exe 的类似物。为此，您需要按照此票据中描述的方式修补 Boost.Build 文件（特别是 tools/build/tools/mc.jam 文件）。之后，您将能够在 b2 中指定 `mc-compiler=windmc` 选项来构建该库。

如果消息编译器检测因某种原因失败，您可以通过在构建库时定义 `BOOST_LOG_WITHOUT_EVENT_LOG` 配置宏来显式禁用事件日志后端支持。这将消除对消息编译器的需求。有关更多配置选项，请参见此部分。

Windows XP 上的 MinGW 用户可能会受到与操作系统捆绑的 msvcrt.dll 中的错误影响。该错误表现为库在格式化日志记录时崩溃。此问题并不特定于 Boost.Log，也可能在与区域设置和 IO 流管理相关的不同上下文中出现。

**针对 Cygwin 用户的附加说明**

Cygwin 支持仍处于初步阶段。Cygwin 中提供的默认 GCC 版本（截至本文撰写为 4.5.3）无法编译该库，因为存在编译器错误。您需要从源代码构建更新的 GCC。即使如此，某些 Boost.Log 功能仍不可用。特别是，基于套接字的 syslog 后端不受支持，因为它依赖于 Boost.ASIO，而该库在该平台上无法编译。然而，本地 syslog 支持仍然存在。

## Configuring and building the library
该库有一个单独编译的部分，应按照“快速入门”指南中的说明进行构建。不过，有一点需要注意。如果您的应用程序由多个模块组成（例如，一个可执行文件和一个或多个 DLL），那么该库必须作为共享对象进行构建。如果您只有一个可执行文件或一个与 Boost.Log 一起工作的模块，则可以将库构建为静态库。

该库支持多种配置宏：
以下是一些宏及其效果：

| 宏名称| 效果  | CMake 说明 |
|------------|---------|--------------|
| **BOOST_LOG_DYN_LINK**  | 如果在用户代码中定义，库将假定二进制文件作为动态加载库构建；<br>否则假定为静态模式。此宏必须在使用日志的所有翻译单元中一致。 | 根据 `BUILD_SHARED_LIBS` CMake 选项自动定义。 |
| **BOOST_ALL_DYN_LINK** | 与 `BOOST_LOG_DYN_LINK` 相同，但也影响其他 Boost 库。|  |
| **BOOST_USE_WINAPI_VERSION**| 影响库和用户代码的编译，选择目标 Windows 版本以提高性能。   | 期望有一个与 `_WIN32_WINNT` 等价的整数值。   |
| **BOOST_LOG_NO_THREADS**    | 如果定义，禁用多线程支持，影响库和用户代码的编译。 | 如果未检测到线程支持，自动定义。    |
| **BOOST_LOG_WITHOUT_CHAR**  | 如果定义，禁用窄字符日志支持，影响库和用户代码的编译。  |  |
| **BOOST_LOG_WITHOUT_WCHAR_T**    | 如果定义，禁用宽字符日志支持，影响库和用户代码的编译。  |  |
| **BOOST_LOG_NO_QUERY_PERFORMANCE_COUNTER** | 仅对 Windows 有用，禁用 `QueryPerformanceCounter` API 的支持。    | 解决与 AMD Athlon CPU 早期版本有关的问题。  |
| **BOOST_LOG_USE_NATIVE_SYSLOG**  | 仅影响库的编译，如果未自动检测到原生 SysLog API 支持，则强制启用。  |  |
| **BOOST_LOG_WITHOUT_DEFAULT_FACTORIES** | 仅影响库的编译，如果定义，则解析器将没有默认的过滤器和格式化器工厂。   |  |
| **BOOST_LOG_WITHOUT_SETTINGS_PARSERS** | 仅影响库的编译，如果定义，则不构建任何与设置解析器相关的设施。    | 禁用 `boost_log_setup` 库的编译。 |
| **BOOST_LOG_WITHOUT_DEBUG_OUTPUT** | 仅影响库的编译，如果定义，Windows 上的调试输出支持将不会构建。    |  |
| **BOOST_LOG_WITHOUT_EVENT_LOG**  | 仅影响库的编译，如果定义，将不构建 Windows 事件日志支持。   | 定义此宏还使消息编译器工具集不再必要。   |
| **BOOST_LOG_WITHOUT_SYSLOG**| 仅影响库的编译，如果定义，将不构建 syslog 后端支持。|  |
| **BOOST_LOG_WITHOUT_IPC**   | 仅影响库的编译，如果定义，将不构建进程间队列支持。  |  |
| **BOOST_LOG_NO_SHORTHAND_NAMES** | 仅影响用户代码的编译，如果定义，一些弃用的简写宏将不可用。  | 不是 CMake 配置选项。|
| **BOOST_LOG_USE_COMPILER_TLS**   | 仅影响库的编译，启用线程局部存储的编译器内部支持。 | 定义可能改善 Boost.Log 的性能，但需要接受某些使用限制。|
| **BOOST_LOG_USE_STD_REGEX**, <br> **BOOST_LOG_USE_BOOST_REGEX**, <br> **BOOST_LOG_USE_BOOST_XPRESSIVE** | 仅影响库的编译，指定 Boost.Log 使用哪种正则表达式库。 | 如果未定义，则默认为 Boost.Regex。使用 `BOOST_LOG_USE_REGEX_BACKEND` 字符串选项代替定义这些宏。 |

您可以通过 b2 命令行定义配置宏，如下所示：

```bash
b2 --with-log variant=release define=BOOST_LOG_WITHOUT_EVENT_LOG define=BOOST_USE_WINAPI_VERSION=0x0600 stage
```

使用 CMake 时，可以在命令行中指定配置宏，如下所示：

```bash
cmake .. -DCMAKE_BUILD_TYPE=Release -DBOOST_LOG_WITHOUT_EVENT_LOG=On
```

不过，在 `boost/config/user.hpp` 文件中定义配置宏可能更方便，这样可以自动为库和用户项目定义它们。如果未指定任何选项，库将尝试支持最全面的设置，包括对所有字符类型和目标平台可用特性的支持。

该日志库还使用了其他几个需要构建的 Boost 库，包括 Boost.Filesystem、Boost.System、Boost.DateTime、Boost.Thread 和在某些配置下的 Boost.Regex。有关构建程序的详细说明，请参阅它们的文档。

最后一点，该库要求在库编译和用户代码编译中启用运行时类型信息（RTTI）。通常，这不需要您做任何事情，只需确保在您的项目中未禁用 RTTI 支持即可。

**关于编译器提供的 TLS 内置特性的一些说明：**

许多常用编译器支持用于管理线程局部存储（TLS）的内置特性，库中的多个部分都使用了这个特性。尽管这些内置特性提供了比 Boost.Thread 或本地操作系统 API 更高效的存储访问，但也有一些注意事项：

1. 某些操作系统（如 Linux 和 Vista 之前的 Windows）在动态加载的共享库中不支持这些内置特性。Windows Vista 及更高版本没有此问题。
2. 在全局构造函数和析构函数中，TLS 访问可能不可靠。例如，Windows 上的 MSVC 8.0 存在此问题。
3. 使用 `BOOST_LOG_USE_COMPILER_TLS` 宏可以启用该特性，从而提升库性能，但需注意：
   - 应用程序可执行文件必须与 Boost.Log 库链接，不能在运行时动态加载。
   - 应用程序不能在全局构造函数或析构函数中使用日志。

同时，启用内置的编译器 TLS 支持并不会消除对 Boost.Thread 或更底层操作系统线程原语的依赖。使用编译器内置 TLS 的目的是为了提升性能，而非减少依赖。

**关于本地 wchar_t 支持的说明：**

一些编译器，尤其是 MSVC，提供禁用本地 `wchar_t` 类型的选项，并用标准整型类型的别名进行模拟。从 C++ 语言的角度来看，这种行为并不符合标准，但对于某些难以更新的遗留代码可能有用。

默认情况下，Boost（包括 Boost.Log）使用启用的本地 `wchar_t`。用户需修改 Boost.Build 以启用模拟模式，尽管可以在该模式下编译 Boost.Log，但要注意以下几点：

1. 编译的 Boost.Log 二进制文件将导出与构建时配置相对应的符号。用户代码必须使用与 Boost.Log 相同的设置，否则会出现链接错误。
2. 在模拟模式下，`wchar_t` 与某个整型类型不可区分，因此库的某些部分可能与本地 `wchar_t` 的正常模式行为不同，宽字符字面量可能会被拒绝或格式化不同。
3. 模拟模式未经测试，可能会出现意外问题。

因此，不建议使用模拟模式，未来版本中可能会完全移除该支持。

**关于 Windows 上 CMake 用户的说明**

为了在 Windows 上使用 CMake 编译带有事件日志支持的 Boost.Log，初始 CMake 配置应确保资源编译工具（rc.exe 或 windres.exe）和消息编译器工具（mc.exe 或 windmc.exe）在 PATH 环境变量中可用。使用 MSVC 时，建议在 Visual Studio 命令行中运行 CMake，或确保设置 Windows SDK 环境变量。

用户还可以设置 RC 和 MC 环境变量为资源和消息编译器可执行文件的路径，或在命令行中设置 `CMAKE_RC_COMPILER` 和 `CMAKE_MC_COMPILER` CMake 选项为相应路径。