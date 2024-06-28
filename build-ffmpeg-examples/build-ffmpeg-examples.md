# 编译 ffmpeg 的 examples 代码

## 简介

ffmpeg 是一个支持对音频流、视频流、字幕流等多媒体流进行解析、处理或转换的一款命令行工具，它可以工作在多媒体文件 container 层面、工作在 stream 层面或者工作在 (decoded) frames 层面，它的代码是面向对象的，它提供了形如 Format（对 Container 的抽象），Packet（对 Stream 的抽象）以及 Frame（对帧的抽象）等抽象概念。

ffmpeg 就其本身而言只是一个命令行前端（命令行工具），它真正的功能实现是通过调用其底层的各个 library 完成的，我们可以通过 `ffmpeg -version` 命令查看 ffmpeg 所赖以工作的这些底层库都有哪些：

```
libavutil      59.  8.100 / 59.  8.100
libavcodec     61.  3.100 / 61.  3.100
libavformat    61.  1.100 / 61.  1.100
libavdevice    61.  1.100 / 61.  1.100
libavfilter    10.  1.100 / 10.  1.100
libswscale      8.  1.100 /  8.  1.100
libswresample   5.  1.100 /  5.  1.100
libpostproc    58.  1.100 / 58.  1.100
```

那么既然如此，理论上，ffmpeg 本身作为一个命令行工具能做到的事情，我们直接编写程序（通过去调用这些库）也能做到。事实上，ffmpeg 官方社区就提供了这样的[例子](https://ffmpeg.org/doxygen/7.0/examples.html)，这个页面也只是提供了一些源代码，但是没有告诉你如何编译它们。

这篇文章的目的是提供一些具体的示例指导读者如何使用 CMake 编译脚本生成工具来对 ffmpeg 官方 API 文档示例代码进行编译、运行。选用 CMake 的好处是跨平台比较容易，它在各个平台上的兼容性都比较好。

## 安装 ffmpeg

我们以开发者常用的操作系统 macOS 系统为工作环境，使用 homebrew (`brew` 命令行) 工具对 ffmpeg（及其底层 library）进行安装，安装完成后，通过 `brew info ffmpeg` 命令查看 ffmpeg 的安装路径：

```sh
$ brew info ffmpeg
Installed
/usr/local/Cellar/ffmpeg/7.0_1 (287 files, 55.7MB) *
```

homebrew 相当于给每一个它安装的 package（在 homebrew 的抽象体系中 package 也叫 keg）都配备了一个“虚拟”的 prefix，对于 ffmpeg@7.0.1 来说，/usr/local/Cellar/ffmpeg/7.0_1 相当于就是它专属的一个虚拟的 prefix，我们如果用 `ls` 命令列举这个目录，就会看到它多少有点类似 `/usr`（大多数 Unix-like 发型版的默认 prefix）：

```sh
$ ls /usr/local/Cellar/ffmpeg/7.0_1
COPYING.GPLv2  COPYING.LGPLv2.1  Changelog	       LICENSE.md  bin	    lib		    share
COPYING.GPLv3  COPYING.LGPLv3	 INSTALL_RECEIPT.json  README.md   include  sbom.spdx.json
```

可以看到其中至少有 bin, lib, share, include 等目录。如果我们对 `ls` 加 `-R` 参数，还会看到该目录的子目录下面还有 `*.h`, `*.pc`, `*.dylib` 等要对 ffmpeg 进行二次开发（实际上是对其底层库对调用）所需要的文件。那么也就是说，既然 homebrew 把 ffmpeg 的这些底层库的头文件和编译配置文件都安装好了，我们写程序调用它们、编译、链接的时候就比较方便了。

## 编译运行 remux.c

`remux.c` 的内容可在[这个页面](https://ffmpeg.org/doxygen/7.0/remux_8c-example.html)找到，我们可以看到它 include 了这两个头文件：

```c
#include <libavutil/timestamp.h>
#include <libavformat/avformat.h>
```

那么至少要编译这个程序，需要告诉编译器 `<libavutil/timestamp.h>` 和 `<libavformat/avformat.h>` 上哪儿找去，因为 homebrew 把没个 keg 安装在单独的 prefix，所以，这些个头文件一般不在默认 prefix 下的 include 目录（比如 /usr/incldue 或者 /usr/local/include）。我们需要 CMake 帮我们做的就是需要它（通过调用 pkg-config）帮我们找到、填充这些必要的包含添加额外 include dir 和额外动态库搜索路径的参数到编译命令里边去。

并且 `remux.c` 还声明、实现了一个 main 函数，这说明 `remux.c` 应当被编译、链接成一个可执行文件而不是又一个 library。

经过这样一番分析，我们现在知道了为什么要用 CMake 或者说我们希望 CMake 能帮我们做什么。

### 编写 CMakeLists.txt

首先初始化一个 CMake 项目目录：

```sh
mkdir build-ffmpeg-examples
cd build-ffmpeg-examples/
touch CMakeLists.txt
```

复制下列内容到 CMakeLists.txt 文件中：

```cmake
cmake_minimum_required(VERSION 3.20)
project(build-ffmpeg-examples)
add_executable(remux remux.c)
```

解释如下：

- `cmake_minimum_required(VERSION 3.20)`：最低要求 CMake 3.20 版本才能通过这个 CMakeLists.txt 生成项目；
- `project(build-ffmpeg-examples)`：项目名是 `build-ffmpeg-examples`；
- `add_executable(remux remux.c)`：创建一个 executable target，这个 target 的名字是 `remux`，它依赖 `remux.c`。

添加这两行指令到 CMakeLists.txt 末尾：

```cmake
include(CMakePrintHelpers)
include(FindPkgConfig)
```

其中 [CMakePrintHelpers](https://cmake.org/cmake/help/latest/module/CMakePrintHelpers.html) 是一个 module，通过 include 这个 module 到我们的项目中，我们可以把这个 module 所提供的一些指令引入到当前作用域中，从而可以使用这些指令，例如 `cmake_print_variables` 指令就是 CMakePrintHelpers module 提供的，顾名思义该指令可以把一个 CMake 变量的变量名和值一同打印到控制台中从而使得调试过程变得简便。

[FindPkgConfig](https://cmake.org/cmake/help/latest/module/FindPkgConfig.html) 也是一个 module，它也提供了一些指令，例如其中比较有用的一个是 `pkg_search_module` 指令，它主要做的是调用 `pkg-config` 可执行文件去搜索要寻找的 `<libname>` 的 `<libname>.pc` pkg-config 配置文件，然后解析里面的信息得到 include dir 和 lib path，然后把这些信息放到一系列给定 prefix 的变量中供我们在 CMakeLists.txt 中去引用、读取。

使用 FindPkgConfig module 之前，你应当确保 `pkg-config` 命令行工具已经安装在系统中，并且可以通过 `$PATH` 找到：

```sh
$ brew install pkg-config
$ which pkg-config
/usr/local/bin/pkg-config
$ pkg-config --version
0.29.2
$ pkg-config --cflags --libs libavutil
-I/usr/local/Cellar/ffmpeg/7.0_1/include -L/usr/local/Cellar/ffmpeg/7.0_1/lib -lavutil
```

通过执行上述命令，我们已经可以（从命令行回显中）确定：

1. pkg-config 命令行工具已经安装；
2. pkg-config 可执行文件的路径、版本；
3. pkg-config 可以找到 libavutil 这个库（那么类似地，ffmpeg 的其他库应该也可以找到，例如 libavformat）；

那么我们可以进入下一步：把这些操作写到 CMakeLists.txt 中以实现自动化：

```cmake
pkg_search_module(AVUTIL REQUIRED libavutil)
cmake_print_variables(AVUTIL_FOUND)
pkg_search_module(AVFORMAT REQUIRED libavformat)
cmake_print_variables(AVFORMAT_FOUND)
```

命令 `pkg_search_module(AVUTIL REQUIRED libavutil)` 把 pkg-config 所能找到的关于 libavutil 库的一些信息填充到一系列以 `AVUTIL_` 开头的 CMake 变量中（`AVUTIL_XXX`），其中就有 `AVUTIL_FOUND`（表示 libavutil 是否找到），`AVUTIL_LIBRARIES`（库对象文件路径），`AVUTIL_INCLUDE_DIRS`（头文件路径）等。

命令 `pkg_search_module(AVFORMAT REQUIRED libavformat)` 类似地（透过 pkg-config 工具）查找 libavformat 库的位置，并且把相应的信息填充到 `AVFORMAT_` 开头的一系列变量中。

FindPkgConfig module 的[文档页面](https://cmake.org/cmake/help/latest/module/FindPkgConfig.html)对于 `pkg_search_module` 都填充哪些变量做了更加详细的介绍。

创建并且将 `remux.c` 的[源码](https://ffmpeg.org/doxygen/7.0/remux_8c-example.html)粘贴到 `remux.c` 文件中，然后运行：

```sh
cmake --fresh -S . -B cmake-build
```

我们可以看到下列输出：

```
-- Checking for one of the modules 'libavutil'
-- AVUTIL_FOUND="1"
-- Checking for one of the modules 'libavformat'
-- AVFORMAT_FOUND="1"
```

这就说明 `pkg_search_module` 指令执行后已成功找到 `libavutil` 库和 `libavformat` 库，相应的以 `AVUTIL_` 和 `AVFORMAT_` 为前缀的一系列 CMake 变量也已经被填充。

在该命令中：

- `--fresh` 表示清空任何已有的 CMake 缓存；
- `-S .` 表示源文件目录处于当前目录，这样 CMake 知道去哪儿寻找 `remux.c`，`CMakeLists.txt` 等文件；
- `-B cmake-build` 表示 CMake 生成产物（编译器脚本、编译参数、以及后期的编译产物等）应该置于（当前目录下的）`cmake-build` 目录。

那么既然 libavformat 和 libavutil 都已找到，我们接下来就可以放下大胆地引用 pkg-config 解析出的相应的编译器、链接器参数，在 CMakeLists.txt 文件中添加下列内容：

```cmake
target_include_directories(remux
    PUBLIC ${AVUTIL_INCLUDE_DIRS}
    PUBLIC ${AVFORMAT_INCLUDE_DIRS}
    )

target_link_libraries(remux
    PUBLIC ${AVUTIL_LINK_LIBRARIES}
    PUBLIC ${AVFORMAT_LINK_LIBRARIES}
    )

cmake_print_properties(TARGETS remux PROPERTIES INCLUDE_DIRECTORIES LINK_LIBRARIES)
```

然后再次运行：

```sh
cmake --fresh -S . -B cmake-build
```

生成项目。

在上述 CMakeLists.txt 文件中，`cmake_print_properties` 指令打印 remux 这个 target 的一些属性，包括 `INCLUDE_DIRECTORIES` 和 `LINK_LIBRARIES`，打印内容在 `cmake` 命令行回显中可以看到：

```
Properties for TARGET remux:
   remux.INCLUDE_DIRECTORIES = "/usr/local/Cellar/ffmpeg/7.0_1/include;/usr/local/Cellar/ffmpeg/7.0_1/include"
   remux.LINK_LIBRARIES = "/usr/local/Cellar/ffmpeg/7.0_1/lib/libavutil.dylib;/usr/local/Cellar/ffmpeg/7.0_1/lib/libavformat.dylib"
```

然后运行：

```
cmake --build cmake-build
```

来执行编译、链接操作。

由于 CMake 默认生成 Makefile 格式的编译器配置，所以，某种程度上，`CMake -S . -B <dir>` 相当于 `./configure`，而 `CMake --build .` 相当于 `make`。

该命令生成的编译目标产物（默认和 target 名一致）位于 `--build` 参数指定的目录下，在这里它的路径也就是 `cmake-build/remux`。

在命令行直接执行：

```sh
$ cmake-build/remux
usage: cmake-build/remux input output
API example program to remux a media file with libavformat and libavcodec.
The output format is guessed according to the file extension.
```

会打印正确用法，并且提示这是一个 API example program，这样，我们就成功编译并且运行了 remux.c 程序。

## 编译其余的例子

这些例子位于 [https://ffmpeg.org/doxygen/7.0/examples.html](https://ffmpeg.org/doxygen/7.0/examples.html)，总共有：

- [avio_http_serve_files.c](https://ffmpeg.org/doxygen/7.0/avio_http_serve_files_8c-example.html)
- [avio_list_dir.c](https://ffmpeg.org/doxygen/7.0/avio_list_dir_8c-example.html)
- [avio_read_callback.c](https://ffmpeg.org/doxygen/7.0/avio_read_callback_8c-example.html)
- [decode_audio.c](https://ffmpeg.org/doxygen/7.0/decode_audio_8c-example.html)
- [decode_filter_audio.c](https://ffmpeg.org/doxygen/7.0/decode_filter_audio_8c-example.html)
- [decode_filter_video.c](https://ffmpeg.org/doxygen/7.0/decode_filter_video_8c-example.html)
- [decode_video.c](https://ffmpeg.org/doxygen/7.0/decode_video_8c-example.html)
- [demux_decode.c](https://ffmpeg.org/doxygen/7.0/demux_decode_8c-example.html)
- [encode_audio.c](https://ffmpeg.org/doxygen/7.0/encode_audio_8c-example.html)
- [encode_video.c](https://ffmpeg.org/doxygen/7.0/encode_video_8c-example.html)
- [extract_mvs.c](https://ffmpeg.org/doxygen/7.0/extract_mvs_8c-example.html)
- [ffhash.c](https://ffmpeg.org/doxygen/7.0/ffhash_8c-example.html)
- [filter_audio.c](https://ffmpeg.org/doxygen/7.0/filter_audio_8c-example.html)
- [hw_decode.c](https://ffmpeg.org/doxygen/7.0/hw_decode_8c-example.html)
- [mux.c](https://ffmpeg.org/doxygen/7.0/mux_8c-example.html)
- [qsv_decode.c](https://ffmpeg.org/doxygen/7.0/qsv_decode_8c-example.html)
- [qsv_transcode.c](https://ffmpeg.org/doxygen/7.0/qsv_transcode_8c-example.html)
- [remux.c](https://ffmpeg.org/doxygen/7.0/remux_8c-example.html)
- [resample_audio.c](https://ffmpeg.org/doxygen/7.0/resample_audio_8c-example.html)
- [scale_video.c](https://ffmpeg.org/doxygen/7.0/scale_video_8c-example.html)
- [show_metadata.c](https://ffmpeg.org/doxygen/7.0/show_metadata_8c-example.html)
- [transcode.c](https://ffmpeg.org/doxygen/7.0/transcode_8c-example.html)
- [transcode_aac.c](https://ffmpeg.org/doxygen/7.0/transcode_aac_8c-example.html)
- [vaapi_encode.c](https://ffmpeg.org/doxygen/7.0/vaapi_encode_8c-example.html)
- [vaapi_transcode.c](https://ffmpeg.org/doxygen/7.0/vaapi_transcode_8c-example.html)

它们中的每一个对于学习 ffmpeg 来说都是很好的参考材料，我认为或许学习 ffmpeg 最佳的方式之一，就是去理解这些代码、修改这些代码然后编译运行查看和验证效果。

这些文件全部在 `$PREFIX/share/ffmpeg/examples` 下可以找到，它甚至还默认附带了一份 Makefile 和 README 文件教你如何编译它们，但是为了更加灵活地将 ffmpeg 底层库集成到我们（将来）自己的项目中，用 CMake 总体来说还是一个更好的选择。我们可以用 `cp` 命令将它们全部复制过来，而不需要一个个去官网都手动复制粘贴一遍。

将 `examples/*.c` 源文件全部复制过来后，用下列脚本自动生成 CMakeLists.txt：

```bash
#!/bin/bash

echo "cmake_minimum_required(VERSION 3.20)"
echo "project(build-ffmpeg-examples)"

echo -ne '\n'
echo "include(CMakePrintHelpers)"
echo "include(FindPkgConfig)"

echo -ne '\n'
for f in $(ls  ./*.c); do
  srcName=$(basename $f .c)
  echo "add_executable($srcName ${srcName}.c)"
  echo -ne '\n'
done

echo -ne '\n'
includeStr=""
linkLibStr=""
for f in $(ls /usr/local/Cellar/ffmpeg/7.0_1/lib/pkgconfig); do
  libname=$(basename $f .pc)
  cmakePrefix=$(tr '[:lower:]' '[:upper:]' <<< $libname)
  cmakePrefix=$(sed -e 's/LIB//g' <<< $cmakePrefix)

  echo "pkg_search_module($cmakePrefix REQUIRED $libname)"
  echo "cmake_print_variables(${cmakePrefix}_FOUND)"
  echo -ne '\n'

  includeStr="$includeStr PUBLIC \${${cmakePrefix}_INCLUDE_DIRS}"
  linkLibStr="$linkLibStr PUBLIC \${${cmakePrefix}_LINK_LIBRARIES}"
done

echo -ne '\n'
for f in $(ls  ./*.c); do
  srcName=$(basename $f .c)
  echo "target_include_directories($srcName$includeStr)"
  echo -ne '\n'
  echo "target_link_libraries($srcName$linkLibStr)"
  echo -ne '\n'
done
```

将它保存为当前目录下的 `gen-cmake.sh`，然后执行

```sh
chmod u+x ./gen-cmake.sh
./gen-cmake.sh > CMakeLists.txt
cmake --fresh -S . -B cmake-build
cmake --build cmake-build
```

就可以编译所有 examples 下的 target 了，成功编译的目标产物将被置于 cmake-build 目录下。

由于依赖库、硬件环境等原因，可能并不是所有的 target 例子都能通过编译，对于编译不通过的 target，可手动在生成的 CMakeLists.txt 中注释掉。比如说笔者就注释掉了 `filter_audio`, `hw_decode`, `mux`, `qsv_decode` 这几个 target.

## 总结与思考

请读者独立思考下列问题，以加深理解：

1. CMake 命令行工具在这里起到什么作用？（换句话说，CMake 它主要干了什么？）
2. pkg-config 命令行工具在这里起到了什么作用？（换句话说，pkg-config 它主要干了什么？）
3. 编译器前端如何寻找头文件？当我们需要在代码中引用一个库的头文件时，怎样让编译器前端「知道」上哪儿去找这些头文件？
4. 从 C 语音编写的源代码文件（简称源文件），到可执行文件，中间大致可分为几个步骤？
5. 从源文件到可执行文件的过程中有一个阶段（或者步骤）叫做「链接」，具体来说链接器需要知道库提供的对象文件（object files）的具体位置，它如何知道？
6. 说说什么是链接？链接过程中链接器主要对什么东西主要做了什么事？
7. 我们用了一个 CMake module 叫做 FindPkgConfig，因为这个 module 的引入使得 CMake 具备了调用 pkg-config 命令行工具去寻找库头文件和对象文件（统称为库文件）的位置的能力，所以 FindPkgConfig 的引入是必须的吗？没有它的情况下，如何让 CMake 顺利地找到库文件的位置？
8. CMake 找到了对象文件的位置后，如何传递给链接器（通过哪条语句）？
9. CMake 可以不传对象文件的具体位置给链接器，而是（只告诉链接器和对象文件有关的一部分线索）让链接器自己去找吗？
10. 为什么有的头文件，即使它不属于 C 标准库，也不需要在 CMakeLists.txt 文件中做额外配置（引入）就能直接在 C 源文件中使用？
11. 假设我手头上只有 ffmpeg 代码库的各种头文件，没有任何它提供的对象文件，我还能顺利编译这些代码例子吗？为什么有的库宣称是 header only 的？
12. 实际上不只有 C 语言经过编译、汇编后可以得到对象文件，是否有这样一种可能，我用其他语言编写源文件，经过编译、汇编后得到的对象文件也能和 ffmpeg 提供的对象文件进行链接得到同样功能且行为正常的可执行文件？对象文件中一般存储着机器码、数据和关于代码、数据、地址、符号的元信息等，严格意义上来说，从对象文件的角度来看它最开始的源文件是由什么高级语言编写的已经不重要了，那么，是什么使得一个对象文件中的机器码能够顺利的调用在另一个对象文件中定义、实现了的函数（过程）？
13. 可执行文件也是一种对象文件，它和链接前的对象文件区别主要在哪？
14. 可执行文件执行后，最初我们在源代码里面定义、实现的 main 函数就一定会被执行吗？理论上来说，是不是可以修改可执行文件的二进制内容，让 main 函数不被执行？
15. 链接成功后，程序正常运行并且得到正确输出，这时，我可以把 ffmpeg 的那些个库文件（主要是对象文件）都给删掉吗？什么条件下可以删掉，什么条件下不可以？
16. 在 C++ 源码中引用 ffmpeg 库函数要注意些什么？
17. 当我已经成功构建、运行了一次引用了 ffmpeg 库函数的程序后，就再也没有改动过这个程序可执行文件的任何内容，可是某一天随着我把系统中安装的 ffmpeg 软件进行版本升级，我再次运行先前的那个软件发现它的行为改变了，这是可能发生的吗（排除编码中存在 bug 的原因）？
