# CMake

> CMake 是一个跨平台的构建系统生成器，不直接构建项目，而是生成对应平台的构建文件。

See also: [[Makefile]], [[C++构建常见问题与技巧]], [[C++语言特性]]

---

## 一、基本概念

**CMake** 是一个跨平台的构建系统生成器，不直接构建项目，而是生成对应平台的构建文件：
- Linux → Makefile
- Windows → Visual Studio 项目文件 / NMake / Ninja
- macOS → Xcode 项目 / Makefile

**核心概念**：
- **CMakeLists.txt**：CMake 的配置文件
- **生成器（Generator）**：生成不同平台的构建文件
- **目标（Target）**：可执行文件、库等构建产物
- **变量（Variable）**：用于配置构建过程

[src: raw/ingested/2技术/cpp/C++构建完整手册-3.-CMake.md]

---

## 二、最小 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.10)
project(MyProject LANGUAGES CXX)

# 设置 C++ 标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 添加可执行文件
add_executable(myapp main.cpp)
```

[src: raw/ingested/2技术/cpp/C++构建完整手册-3.-CMake.md]

---

## 三、常用命令

```cmake
# 项目设置
cmake_minimum_required(VERSION 3.10)
project(ProjectName [LANGUAGES] CXX)

# 设置变量
set(VAR_NAME value)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")

# 添加可执行文件
add_executable(target_name source1.cpp source2.cpp)

# 添加库
add_library(lib_name STATIC source1.cpp source2.cpp)   # 静态库
add_library(lib_name SHARED source1.cpp source2.cpp)   # 动态库
add_library(lib_name INTERFACE)                         # 接口库（仅头文件）

# 包含目录
target_include_directories(target_name PUBLIC include/)
target_include_directories(target_name PRIVATE src/)

# 链接库
target_link_libraries(target_name PRIVATE lib_name)
target_link_libraries(target_name PUBLIC lib_name)

# 查找包
find_package(Boost REQUIRED COMPONENTS system filesystem)

# 选项
option(ENABLE_TESTS "Enable tests" ON)
if(ENABLE_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()
```

[src: raw/ingested/2技术/cpp/C++构建完整手册-3.-CMake.md]

---

## 四、多文件与多目录项目

### 目录结构

```
project/
├── CMakeLists.txt          # 根 CMakeLists.txt
├── src/
│   ├── CMakeLists.txt
│   └── main.cpp
├── include/
│   └── mylib.h
├── lib/
│   ├── CMakeLists.txt
│   └── mylib.cpp
└── tests/
    ├── CMakeLists.txt
    └── test.cpp
```

### 根 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.10)
project(MyProject VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)

# 设置输出目录
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

# 添加子目录
add_subdirectory(lib)
add_subdirectory(src)

if(BUILD_TESTING)
    enable_testing()
    add_subdirectory(tests)
endif()
```

### 库的 CMakeLists.txt

```cmake
add_library(mylib STATIC mylib.cpp)

target_include_directories(mylib
    PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/../include
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include>
        $<INSTALL_INTERFACE:include>
)

target_compile_features(mylib PUBLIC cxx_std_17)
```

### 可执行文件的 CMakeLists.txt

```cmake
add_executable(myapp main.cpp)
target_link_libraries(myapp PRIVATE mylib)
```

[src: raw/ingested/2技术/cpp/C++构建完整手册-3.-CMake.md]

---

## 五、目标属性与作用域

### 目标属性

```cmake
# 设置编译选项
target_compile_options(target_name PRIVATE -Wall -Wextra)

# 设置编译定义
target_compile_definitions(target_name PRIVATE DEBUG_MODE)

# 设置 C++ 标准
set_target_properties(target_name PROPERTIES
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED ON
    CXX_EXTENSIONS OFF
)

# 设置输出名称
set_target_properties(target_name PROPERTIES
    OUTPUT_NAME "custom_name"
    PREFIX ""
    SUFFIX ".exe"
)
```

### PUBLIC、PRIVATE、INTERFACE

```cmake
# PUBLIC：当前目标和依赖它的目标都能使用
target_include_directories(mylib PUBLIC include/)

# PRIVATE：只有当前目标使用
target_include_directories(mylib PRIVATE src/)

# INTERFACE：只有依赖它的目标使用，当前目标不使用
target_include_directories(mylib INTERFACE include/)

# target_link_libraries 同理
target_link_libraries(mylib
    PRIVATE dep1    # 仅 mylib 使用
    PUBLIC dep2     # mylib 使用，也传递给依赖 mylib 的目标
    INTERFACE dep3  # mylib 不使用，但传递给依赖 mylib 的目标
)
```

[src: raw/ingested/2技术/cpp/C++构建完整手册-3.-CMake.md]

---

## 六、查找和使用外部库

### find_package

```cmake
# 查找 Boost
find_package(Boost 1.70 REQUIRED COMPONENTS system filesystem thread)
if(Boost_FOUND)
    target_link_libraries(myapp PRIVATE 
        Boost::system Boost::filesystem Boost::thread
    )
endif()

# 查找 OpenSSL
find_package(OpenSSL REQUIRED)
target_link_libraries(myapp PRIVATE OpenSSL::SSL OpenSSL::Crypto)

# 查找 Threads
find_package(Threads REQUIRED)
target_link_libraries(myapp PRIVATE Threads::Threads)
```

### pkg-config

```cmake
find_package(PkgConfig REQUIRED)
pkg_check_modules(GTK3 REQUIRED gtk+-3.0)
target_link_libraries(myapp PRIVATE ${GTK3_LIBRARIES})
target_include_directories(myapp PRIVATE ${GTK3_INCLUDE_DIRS})
```

### 手动指定路径

```cmake
set(MYLIB_DIR /usr/local/lib)
set(MYLIB_INCLUDE_DIR /usr/local/include)
target_include_directories(myapp PRIVATE ${MYLIB_INCLUDE_DIR})
target_link_directories(myapp PRIVATE ${MYLIB_DIR})
target_link_libraries(myapp PRIVATE mylib)
```

### find_package 的 CONFIG 和 MODULE 模式

```cmake
# MODULE 模式：使用 FindXXX.cmake 模块
find_package(OpenSSL MODULE)

# CONFIG 模式：使用 XXXConfig.cmake 配置文件
find_package(Qt5 CONFIG)

# 默认先找 MODULE，再找 CONFIG
find_package(Boost)
```

[src: raw/ingested/2技术/cpp/C++构建完整手册-3.-CMake.md]

---

## 七、生成器表达式

```cmake
# 条件编译
target_compile_definitions(mylib PRIVATE
    $<$<CONFIG:Debug>:DEBUG_MODE>
    $<$<CONFIG:Release>:NDEBUG>
)

# 平台相关
target_compile_definitions(mylib PRIVATE
    $<$<PLATFORM_ID:Windows>:WIN32>
    $<$<PLATFORM_ID:Linux>:LINUX>
)

# 编译器相关
target_compile_options(mylib PRIVATE
    $<$<CXX_COMPILER_ID:MSVC>:/W4>
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall -Wextra>
)
```

[src: raw/ingested/2技术/cpp/C++构建完整手册-3.-CMake.md]

---

## 八、条件编译与选项

```cmake
# 定义选项
option(ENABLE_DEBUG "Enable debug mode" OFF)
option(BUILD_TESTS "Build test executables" ON)

if(ENABLE_DEBUG)
    target_compile_definitions(myapp PRIVATE DEBUG_MODE)
endif()

# 条件添加源文件
if(WIN32)
    target_sources(myapp PRIVATE src/platform_win.cpp)
elseif(UNIX)
    target_sources(myapp PRIVATE src/platform_unix.cpp)
endif()
```

[src: raw/ingested/2技术/cpp/C++构建完整手册-3.-CMake.md]

---

## 九、生成配置头文件

```cmake
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/config.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/config.h
)
target_include_directories(myapp PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
```

**config.h.in:**
```c
#ifndef CONFIG_H
#define CONFIG_H
#define PROJECT_VERSION_MAJOR @PROJECT_VERSION_MAJOR@
#define PROJECT_NAME "@PROJECT_NAME@"
#cmakedefine ENABLE_FEATURE_X
#cmakedefine01 USE_OPENSSL
#endif
```

[src: raw/ingested/2技术/cpp/C++构建完整手册-3.-CMake.md]

---

## 十、构建类型

```cmake
# 设置默认构建类型
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

# 根据构建类型设置选项
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    target_compile_definitions(mylib PRIVATE DEBUG)
    target_compile_options(mylib PRIVATE -g -O0)
elseif(CMAKE_BUILD_TYPE STREQUAL "Release")
    target_compile_definitions(mylib PRIVATE NDEBUG)
    target_compile_options(mylib PRIVATE -O3 -DNDEBUG)
endif()
```

**构建类型**：Debug、Release、RelWithDebInfo、MinSizeRel

[src: raw/ingested/2技术/cpp/C++构建完整手册-3.-CMake.md]

---

## 十一、测试

```cmake
enable_testing()

add_test(NAME MyTest COMMAND myapp --test)

add_executable(test_runner test.cpp)
target_link_libraries(test_runner PRIVATE mylib)
add_test(NAME UnitTests COMMAND test_runner)
```

[src: raw/ingested/2技术/cpp/C++构建完整手册-3.-CMake.md]

---

## 十二、常用变量

```cmake
# 项目信息
${CMAKE_PROJECT_NAME}        # 项目名称
${PROJECT_VERSION}            # 项目版本
${CMAKE_SOURCE_DIR}           # 源码根目录
${CMAKE_BINARY_DIR}           # 构建根目录
${CMAKE_CURRENT_SOURCE_DIR}   # 当前 CMakeLists.txt 所在目录

# 编译器和工具
${CMAKE_CXX_COMPILER}         # C++ 编译器
${CMAKE_BUILD_TYPE}           # Debug/Release/RelWithDebInfo/MinSizeRel

# 平台信息
${CMAKE_SYSTEM_NAME}          # Linux/Windows/Darwin
${WIN32} / ${UNIX} / ${APPLE} # 平台宏
```

[src: raw/ingested/2技术/cpp/C++构建完整手册-3.-CMake.md]

---

## Related Pages
- [[Makefile]]
- [[C++构建常见问题与技巧]]
- [[C++语言特性]]
- [[C++第三方库手册]]
