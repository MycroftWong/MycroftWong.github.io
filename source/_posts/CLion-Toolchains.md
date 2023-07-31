---
title: CLion Toolchains
date: 2023-07-26 17:46:56
tags:
---

# CLion Toolchains

`Toolchains` 工具链，一些列工具。

`ChatGPT` 解答 `Toolchains` 是什么：

> 在 CLion 中，Toolchain 是指编译和构建项目所需的一组工具和配置。它包括编译器、链接器、构建系统等，用于将源代码转换为可执行文件或库文件。
> 
> CLion 支持多种不同的 Toolchain，您可以根据项目的需求选择合适的 Toolchain。

简单来说，`Toolchain` 包含编译、链接、构建、调试等的一系列工具。在 `CLion` 的配置中我们可以看到包括 `CMake`、`Build Tool`、`C Compiler`、`C++ Compiler`、`Debugger`。

在不同的操作系统上，这些工具会有一些差异，如在 `Linux` 中，`C Compiler` 通常为 `cc`，`C++ Compiler` 通常为 `c++`，`Debugger` 为 `gdb`。而在 `Mac` 上，默认的 `Debugger` 则为 `LLDB`

## C Compiler

询问 `ChatGPT` 有哪些 `C Compiler`：

> 在C语言编程中，有多种C编译器可供选择，这些编译器可以将C源代码转换为可执行的机器代码。以下是一些常见的C编译器：
> 
> 1. GCC (GNU Compiler Collection)：GCC 是一个广泛使用的开源C编译器，它支持多种平台和架构，并且拥有强大的优化功能。GCC是许多Linux发行版和其他类Unix系统中默认的C编译器。
> 
> 2. Clang：Clang 是LLVM项目的一部分，是一个快速、高度兼容C编译器。它在性能上与GCC相媲美，并且支持较新的C标准。
> 
> 3. Microsoft Visual C++：Microsoft Visual C++ 是Windows平台上的官方C编译器，它集成在Visual Studio开发环境中。它通常用于Windows平台的C语言开发。
> 
> 4. Intel C++ Compiler：Intel C++ Compiler 是由英特尔公司开发的C编译器，专注于优化性能和并行计算。
> 
> 5. TCC (Tiny C Compiler)：TCC 是一个小巧、快速的C编译器，适用于嵌入式系统和资源受限的环境。
> 
> 6. Pelles C：Pelles C 是一个Windows平台上的免费C编译器，适用于学习和轻量级C开发。
> 
> 7. Open Watcom：Open Watcom 是一个开源的C/C++编译器套件，支持多种平台。
> 
> 这些是一些常见的C编译器，每个编译器都有其特定的优点和用途。您可以根据需要和平台选择合适的C编译器来进行C语言开发。

简而言之，有多种 `C Compiler`，每种操作系统有自己特有的 `C Compiler`，如 `Linux` 上的 `GCC`，`MacOS` 上的 `Clang`，`Windows` 上的 `Microsoft Visual C++`。

而 `Microsoft Visual C++` 实际上是 `C++ Compiler`，它支持 `C90` 标准，也在逐步支持 `C99`，但未完全支持 `C99` 标准。

### GCC

> GCC (GNU Compiler Collection) 是一个广泛使用的开源编译器集合，由GNU计划开发。它支持多种编程语言，包括C、C++、Objective-C、Fortran、Ada等，并能在多种操作系统和架构上运行。
> 
> GCC 是一个强大的编译器，它不仅能将源代码编译成可执行的机器代码，还提供了许多优化选项和功能，以提高程序的性能和效率。GCC 还支持各种平台上的交叉编译，可以为不同的目标平台生成代码。
> 
> GCC 是自由软件，使用GNU通用公共许可证 (GPL) 授权，这意味着它可以免费使用，并且用户可以查看和修改其源代码。由于其功能强大和广泛的支持，GCC 成为了许多Linux发行版和其他类Unix系统中的默认编译器，也被广泛应用于各种开发项目中。

`GCC` 实际上是 `GNU` 计划开发的 `C Compiler`，基本很快支持最新的 `C` 标准。

### Clang

> Clang 是一个开源的C、C++、Objective-C 和Objective-C++编程语言的编译器，它是 LLVM（Low Level Virtual Machine）项目的一部分。Clang 最初由苹果公司开发，现在由一个全球性的开发社区维护和发展。
> 
> Clang 的目标是成为一个高性能、高质量的编译器，它主要关注于编译速度、代码质量和用户友好性。Clang 的设计目标之一是提供更好的错误和警告信息，以帮助开发者更容易地发现和解决代码中的问题。
> 
> Clang 支持多种平台和架构，可以在 Linux、macOS、Windows 和其他操作系统上运行。它在性能上与传统的编译器（如GCC）相媲美，并且有一些独特的优势，如更好的错误诊断、更快的编译速度和更好的模板支持等。
> 
> 除了作为独立的编译器使用，Clang 还被广泛应用于其他项目中，如编码工具、静态代码分析工具、代码编辑器等。在 macOS 和 iOS 平台上，Xcode 集成了 Clang 作为默认的编译器。
> 
> 总的来说，Clang 是一个强大的、高质量的开源编译器，它为 C、C++ 和 Objective-C 等编程语言提供了一种快速、高质量的编译解决方案，并且得到了全球开发者的广泛支持和贡献。

`Clang` 实际上就是 `LLVM` 项目中开发的 `C`、`C++`、`Objective-C`、`Objective-C++` 编译器。虽然可以在多种系统上运行，但是实际上一般只会在 `MacOS` 上使用。

#### LLVM - Low Level Virtual Machine

> LLVM 是一个开源的编译器基础设施项目，它最初是由苹果公司开发的，现在由一个全球性的开发社区维护和发展。LLVM 的名称最初是 "Low Level Virtual Machine" 的缩写，但随着项目的发展，它已经成为一个代表更广泛目标的名字。
> 
> LLVM 提供了一套模块化、灵活和可扩展的编译器工具和技术，用于编译各种编程语言的源代码到可执行的机器代码。它的设计目标是提供高度优化的编译能力，同时提供灵活性和可扩展性，使得开发者能够针对不同的目标平台进行代码优化。
> 
> LLVM 的核心组件包括：
> 
> 1. 前端 (Frontend)：负责将不同编程语言（如C、C++、Objective-C、Swift等）的源代码转换成中间表示（Intermediate Representation，IR）。
> 
> 2. 中间表示 (IR)：LLVM使用一种通用的中间表示，它是一种低级别的、静态单赋值（Static Single Assignment，SSA）形式的代码表示形式。这种IR使得LLVM能够在不同的优化阶段进行代码优化。
> 
> 3. 优化器 (Optimizer)：负责在中间表示层次上对代码进行各种优化，以提高代码的性能和效率。
> 
> 4. 后端 (Backend)：负责将优化后的中间表示代码转换成目标平台的机器代码。
> 
> LLVM 的设计和实现使得它能够支持多种编程语言，能够在不同的硬件和操作系统上进行交叉编译，也能够应用于各种领域，如编译器、调试器、虚拟机等。
> 
> LLVM 的开源许可证是 Apache 2.0 License，这意味着任何人都可以免费使用、修改和分发LLVM的代码。LLVM 的开发社区非常活跃，它得到了全球开发者的广泛支持和贡献。

`LLVM` 实际就是一系列的编译工具。将源代码经过多重处理后，生成目标平台的机器代码。

前两天刚好看到 `Bennyhuo` 老师对 `kotlin` 的科普，`kotlin` 的编译器实际上也是使用 `kotlin` 编写的，目前 `kotlin` 稳定版本是 `1.9.0`，但 `kotlin` 后端编译部分已经在使用 `k2（kotlin 2.0）` 了。

### Microsoft Visual C++

> Microsoft Visual C++（简称VC++）是微软公司开发的一款集成开发环境 (IDE) 和 C++ 编译器，主要用于在 Windows 平台上进行 C++ 开发。它是 Microsoft Visual Studio 的一部分，提供了丰富的功能和工具，用于编写、调试和部署 C++ 应用程序。
> 
> Microsoft Visual C++ 提供了一个直观的图形用户界面，集成了代码编辑器、调试器、资源编辑器、可视化设计器等工具，使得开发者可以在一个集成的开发环境中完成整个 C++ 开发流程。
> 
> 主要特点和功能包括：
> 
> 1. C++ 编译器：Microsoft Visual C++ 提供了一个强大的 C++ 编译器，用于将 C++ 源代码编译成可执行的机器代码。
> 
> 2. 代码编辑器：VC++ 的代码编辑器提供了代码自动补全、语法高亮、代码折叠等功能，帮助开发者提高编码效率。
> 
> 3. 调试器：VC++ 的调试器支持逐步调试、断点调试、变量监视等功能，帮助开发者定位和修复程序中的错误。
> 
> 4. 可视化设计器：VC++ 提供了可视化设计器，使得开发者可以通过拖拽和放置的方式设计用户界面。
> 
> 5. 支持多种工程类型：VC++ 支持多种工程类型，包括控制台应用程序、Windows 窗体应用程序、DLL 库等。
> 
> 6. 广泛的库和工具：VC++ 提供了丰富的库和工具，包括标准模板库（STL）、MFC（Microsoft Foundation Classes）、ATL（Active Template Library）等，帮助开发者快速构建应用程序。
> 
> Microsoft Visual C++ 是一个强大且广泛使用的 C++ 开发工具，适用于 Windows 平台上的各种 C++ 应用程序开发，从简单的控制台程序到复杂的图形界面应用程序和游戏开发都可以使用 VC++ 来完成。

`Microsoft Visual C++` 更多的是对 `C++` 的支持，对 `C` 的支持并不是很好。

所以我们在 `Windows` 上进行 `C` 开发时，如果想要使用更高的 `C` 标准，可以安装 `Cygwin` 或 `MinGW`。他们可以在 `Windows` 提供 `GNU` 工具集，如 `GCC`、`GDB`。

`Cygwin` 和 `MinGW` 的目的是在 `Windows` 上提供 `GNU` 工作环境。而不用必须在 `Unix Like` 上工作。实际上，`Windows` 能够提供更全面的开发工具，并且开发的程序也可能需要运行在 `Windows` 上，所以使用 `Cygwin`、`MinGW` 能够很好的解决 `Microsoft Visual C++` 对 `C` 标准的支持。

### Cygwin

> Cygwin 是一个在 Windows 操作系统上提供类 Unix 环境的兼容层和工具集。它允许在 Windows 平台上运行类 Unix 的应用程序和工具，使得开发者可以在 Windows 上编译和运行 Unix/Linux 软件。
> 
> Cygwin 包含了一套开发工具和库，如 GCC（GNU Compiler Collection）、GDB（GNU Debugger）、GNU Make 等，以及一些常用的 Unix/Linux 命令行工具和库，如 ls、grep、awk、sed 等。这些工具和库在 Cygwin 环境下运行，提供了类 Unix 的命令行接口和功能，使得开发者可以在 Windows 上使用类 Unix 的开发环境。
> 
> Cygwin 是自由软件，使用 GNU 通用公共许可证（GPL）许可证，这意味着用户可以免费使用和修改它。Cygwin 的目标是提供一个完整的 Unix/Linux 兼容环境，使得开发者可以在 Windows 上轻松地进行 Unix/Linux 软件的开发和移植。
> 
> 需要注意的是，Cygwin 是一个完整的 Unix/Linux 兼容环境，它在 Windows 上运行的应用程序和工具将会在 Cygwin 环境中运行，而不是原生的 Windows 环境。这意味着 Cygwin 程序依赖于 Cygwin DLL，而不能直接使用 Windows DLL。如果需要在 Windows 上开发本地的 Windows 应用程序，可以考虑使用其他 Windows 开发环境，如 Microsoft Visual Studio 和 MinGW。

### MinGW

> MinGW（Minimalist GNU for Windows）是一个在 Windows 平台上使用 GNU 工具集的开发环境。它提供了一套开发工具和库，使得开发者可以在 Windows 上编译和运行类似于 Unix 系统上的软件。
> 
> MinGW 包含了 GNU 工具链，如 GCC（GNU Compiler Collection）、GDB（GNU Debugger）、GNU Make 等，以及一些常用的开发库，如头文件和静态链接库，使得开发者能够编写和构建在 Windows 上运行的 C、C++ 等应用程序。
> 
> MinGW 是自由软件，使用 GPL 许可证，这意味着用户可以免费使用和修改它。MinGW 的目标是提供一个轻量级的、简化的 GNU 工具链，用于在 Windows 上进行开发，而不需要像 Cygwin 那样提供完整的 Unix 兼容性。
> 
> 由于 MinGW 提供了与 Unix 环境类似的开发工具和库，使得开发者可以更容易地将他们的 Unix/Linux 软件移植到 Windows 平台上，并且可以在 Windows 上开发跨平台的软件。同时，MinGW 也能够开发本地的 Windows 应用程序，这使得它成为一个受欢迎的 Windows 开发环境选择。
> 
> 需要注意的是，MinGW 是一个 32 位的工具集，它不能直接用于开发 64 位的应用程序。对于 64 位开发，可以使用 MinGW-w64，它是 MinGW 的一个分支，提供了对 64 位开发的支持。

### Cygwin vs MinGW

> Cygwin 和 MinGW 都是用于在 Windows 平台上提供类 Unix 环境的开发工具和库，但它们在实现和用途上有一些区别。下面是 Cygwin 和 MinGW 的主要区别：
> 
> 1. 实现方式：
>    - Cygwin：Cygwin 是一个完整的 Unix/Linux 兼容环境，它通过提供一个兼容层来在 Windows 上模拟 Unix/Linux 环境。Cygwin 应用程序和工具在运行时依赖于 Cygwin DLL，它负责处理与 Windows 系统的交互，并提供类 Unix 的环境支持。
>    - MinGW：MinGW 是一个最小化的 GNU 工具链，它提供了一套纯粹的编译器和工具，用于在 Windows 上编译和运行类 Unix 的应用程序。MinGW 通过直接将源代码编译为 Windows 可执行文件来实现，不需要类 Unix 的兼容层。
> 
> 2. 应用场景：
>    - Cygwin：Cygwin 主要用于在 Windows 上运行和开发 Unix/Linux 软件，它提供了完整的 Unix/Linux 兼容环境，可以运行大部分 Unix/Linux 命令行工具和应用程序。Cygwin 可以用于开发跨平台的软件，并且提供了类 Unix 的命令行界面和功能。
>    - MinGW：MinGW 主要用于在 Windows 上开发本地的 Windows 应用程序，它提供了 Windows 版本的 GNU 工具链，可以将 C/C++ 代码编译为原生的 Windows 可执行文件。MinGW 不提供类 Unix 的环境支持，它适用于开发原生的 Windows 应用程序。
> 
> 3. 依赖性：
>    - Cygwin：Cygwin 应用程序和工具依赖于 Cygwin DLL，这意味着在运行 Cygwin 程序时需要有 Cygwin 环境。
>    - MinGW：MinGW 应用程序和工具不依赖于特定的 DLL，它们是原生的 Windows 可执行文件，可以直接在 Windows 上运行。
> 
> 综上所述，Cygwin 和 MinGW 在实现和用途上有所区别。如果需要在 Windows 上运行和开发类 Unix 的应用程序，可以选择 Cygwin；如果需要在 Windows 上开发本地的 Windows 应用程序，可以选择 MinGW。

并没有理解 `ChatGPT` 的解释。下面试试 `Claude 2`：

> Mingw和Cygwin都是在Windows上运行Linux/Unix软件的解决方案,主要区别如下:
> 
> 1. 原理不同:
> 
> - Mingw是最小化的Unix环境和工具集,直接调用Windows的API。
> 
> - Cygwin提供了类Unix的系统调用接口,需要Cygwin DLL进行系统调用转化。
> 
> 2. 目标不同:
> 
> - Mingw主要面向开发者,用于在Windows上编译Linux/Unix程序。
> 
> - Cygwin更像是一个完整的Linux环境,面向所有Linux/Unix用户和程序。
> 
> 3. 性能不同:
> 
> - Mingw编译的程序不依赖DLL,性能更好。
> 
> - Cygwin依赖Cygwin DLL,性能较差。
> 
> 4. 兼容性不同:
> 
> - Mingw编译的程序和Linux二进制更兼容。
> 
> - Cygwin提供的是模拟层,和Linux有差异。
> 
> 5. 使用不同:
> 
> - Mingw更适合交叉编译开发。
> 
> - Cygwin更像是Linux subsystem,可以直接运行Linux程序。
> 
> 总体来说,Mingw适合在Windows上编译Linux/Unix程序,Cygwin适合在Windows上模拟Linux环境。

虽然也不完全明白，但是能够简单理解了。

## Debugger

> C 语言的调试器有很多选择，以下是一些常见的 C 调试器：
> 
> 1. GDB（GNU Debugger）：GDB 是一个功能强大的开源调试器，属于 GNU 工具集的一部分。它支持多种编程语言，包括 C 语言，可以在命令行中使用，也可以与许多集成开发环境（IDE）集成。
> 
> 2. LLDB：LLDB 是一个开源调试器，属于 LLVM 项目的一部分。它最初是为 macOS 和 iOS 开发的，后来也被移植到其他平台上。LLDB 支持多种编程语言，包括 C 语言，具有现代化的调试功能和用户友好的界面。
> 
> 3. Visual Studio Debugger：Visual Studio 是微软公司的集成开发环境（IDE），它包含了一个强大的调试器用于 C 语言开发。Visual Studio Debugger 提供了丰富的调试功能，包括逐步调试、断点调试、变量监视等。
> 
> 4. Eclipse CDT Debugger：Eclipse 是一个开源的跨平台 IDE，CDT（C/C++ Development Tooling）是 Eclipse 的 C/C++ 开发工具包。Eclipse CDT Debugger 提供了调试功能，支持多种编程语言，包括 C 语言。
> 
> 5. Xcode Debugger：Xcode 是苹果公司的集成开发环境（IDE），用于开发 macOS 和 iOS 应用程序。Xcode Debugger（也称为 LLDB）提供了强大的调试功能，支持 C 语言等多种编程语言。
> 
> 这些调试器都提供了丰富的调试功能，开发者可以根据自己的需求和偏好选择合适的调试器来进行 C 语言的调试工作。

可以看到 `GNU`、`LLVM`、`Visual Studio` 都提供了相应的调试器。

## CLion Cmake 配置

就像 `Intellij Idea` 实际上仍然使用 `javac` 编译 `Java` 代码一样，`CLion` 也是使用这些工具对 `C` 代码进行编译、调试的。

> 在 CLion 中配置 Toolchains（工具链）和 CMake 可以让你为你的 C++ 项目选择适当的编译器和构建工具。这样，你可以确保你的项目能够正确编译和构建。
> 
> 以下是在 CLion 中配置 Toolchains 和 CMake 的步骤：
> 
> 配置 Toolchains：
> 1. 打开 CLion，进入 "File"（文件）菜单，然后选择 "Settings"（设置）。
> 2. 在左侧的面板中，选择 "Build, Execution, Deployment"（构建、执行、部署）选项。
> 3. 在右侧的面板中，选择 "Toolchains" 选项卡。
> 4. 点击 "+ Add" 按钮，选择你想要配置的工具链类型，比如 "MinGW"、"MSVC"（Microsoft Visual C++）或者 "WSL"（Windows Subsystem for Linux）。
> 5. 根据你选择的工具链类型，配置相应的工具链路径和参数。例如，对于 MinGW，你需要指定 MinGW 安装路径，对于 MSVC，你需要选择 Visual Studio 安装路径等。
> 6. 点击 "Apply"（应用）按钮保存配置。
> 
> 配置 CMake：
> 1. 在同一个 "Settings" 窗口中，仍然在 "Build, Execution, Deployment" 选项下，选择 "CMake" 选项卡。
> 2. 确保 "CMake executable"（CMake 可执行文件）字段指向正确的 CMake 可执行文件路径。你可以手动输入路径或者使用 "..." 按钮浏览选择。
> 3. 如果你的项目有自己的 CMake 配置文件（CMakeLists.txt），你可以在 "CMake options"（CMake 选项）字段中添加任何你想要传递给 CMake 的参数。这些参数会影响项目的构建过程。
> 4. 点击 "Apply" 按钮保存配置。
> 
> 完成上述配置后，CLion 将根据你选择的工具链和 CMake 配置来构建和编译你的项目。确保你的工程设置和 CMakeLists.txt 文件正确配置，这样你就能顺利构建和运行你的 C++ 项目了。

在配置 `CMake` 时有一个 `Toolchain` 选项，就可以选择我们之前设置的 `Toolchain`，甚至直接可以选择 `Remote`，使用 `ssh` 使用服务器进行编译。

