
目录：
- [编译和链接的过程](#编译和链接的过程)
- [构建系统Makefile](#构建系统makefile)
- [跨平台构建系统CMake](#跨平台构建系统cmake)
  - [多文件项目示例](#多文件项目示例)
  - [现代CMake的一个最佳实践](#现代cmake的一个最佳实践)
- [参考资料](#参考资料)


## 编译和链接的过程

1. 预处理
主要处理一些预处理指令，如`#include`，`#define`，`#ifndef`等，同时删除注释，添加行号和文件名标识。预处理后生成 xxx.i 文件。

2. 编译
编译是将预处理文件转换成汇编代码文件（xxx.s 文件）的过程，具体的步骤主要有：词法分析 -> 语法分析 -> 语义分析及相关的优化 -> 中间代码生成 -> 目标代码生成。

3. 汇编
汇编是将汇编代码文件转换成可重定位目标文件（xxx.o 文件）的过程，目标文件中存放着机器指令，但是还没有经过链接，所以不能直接运行。
Linux系统的目标文件格式是ELF（Executable and Linkable Format）格式。

4. 链接
链接阶段就是把若干个可重定位目标文件整合成一个可执行文件的过程，分为静态链接和动态链接。

## 构建系统Makefile

我们初学C/C++时，大多采用单文件编译的方式，即直接使用编译器编译单个源文件。但是当项目规模较大时，需要编译多个源文件，一个一个地使用gcc/g++进行编译就会很麻烦，这时就需要用到构建系统，如Makefile。

> Makefile有以下几点好处：
> - 更新其中一个源文件时，只需重新编译这一个文件，而不是所有文件
> - 可以同时编译多个文件，提高编译效率
> - 用通配符批量生成构建规则
>
> 也有缺点：
> - 在类Unix系统上通用，但在Windows系统上则不然
> - 需要准确地指明每个项目之间的依赖关系
> - 语法过于简单，不能执行判断的逻辑
> - 不同的编译器有不同的flag规则


## 跨平台构建系统CMake

为了解决Makefile的缺点，CMake应运而生。CMake是一个跨平台的构建系统，它使用CMakeLists.txt文件来描述构建规则，然后生成Makefile或Project文件，再由此进行编译。

### 多文件项目示例

一个简单的示例项目，目录如下：
<img src="./images/cmake-learn/项目一的目录结构.png" width="400">

项目根目录下的CMakeLists.txt文件如下：
```cmake
cmake_minimum_required(VERSION 2.8)

project(demo)

add_subdirectory(src)
```

src目录下的CMakeLists.txt文件如下：
```cmake
aux_source_directory(. SRC_LIST)

include_directories(../include)

add_executable(main ${SRC_LIST})

set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
```

构建该项目：
<img src="./images/cmake-learn/构建项目一.png" width="700">

### 现代CMake的一个最佳实践

现代CMake（通常指3.0+版本）的核心思想是以目标（Target）为中心。与旧式全局设置（如 `include_directories`、`link_directories`）不同，现代CMake使用 `target_***` 系列命令，将属性（如头文件路径、链接库、编译选项）精确关联到具体目标，避免了依赖泄漏和命名冲突。
```cmake
# 旧式（不推荐）
include_directories(./include) # 全局添加，所有目标都受影响
link_directories(./lib)
add_executable(myapp main.cpp)
target_link_libraries(myapp mylib)

# 现代（推荐）
add_executable(myapp main.cpp)
# 属性精确绑定到`myapp`，`PUBLIC`表示依赖者也需要此头文件路径
target_include_directories(myapp PUBLIC ./include)
# 直接链接库目标，CMake会自动查找路径
target_link_libraries(myapp PRIVATE mylib)
```


## 参考资料
[1] CSDN博客《【C++】Cmake使用教程（看这一篇就够了）》
https://blog.csdn.net/weixin_43717839/article/details/128032486?ops_request_misc=elastic_search_misc&request_id=ef002f7d780970a2fe7074984b092323&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_positive~default-1-128032486-null-null.142^v102^pc_search_result_base2&utm_term=cmake%E4%BD%BF%E7%94%A8%E6%95%99%E7%A8%8B&spm=1018.2226.3001.4187
[2] BiliBili视频《【录播】现代C++中的高性能并行编程与优化：学C++从CMake学起》
https://www.bilibili.com/video/BV1fa411r7zp/?spm_id_from=333.1391.0.0
[3] 2025-Yatcpu-Fall-Docs: 编译与链接&CMake 
https://purplepower.github.io/2025-fall-yatcpu-repo/better-tut/practice/compile-and-link.html
[4] 《深入理解计算机系统（CSAPP）》第七章：链接