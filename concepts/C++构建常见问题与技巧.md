# C++ 构建常见问题与技巧

See also: [[C++语言特性]], [[C++第三方库手册]]

## 一、常见错误

### 找不到头文件
```cmake
target_include_directories(target_name PRIVATE include/)
```

### 链接错误
```cmake
target_link_libraries(target_name PRIVATE lib1 lib2 pthread)
```

### 库路径问题
```cmake
target_link_directories(target_name PRIVATE /path/to/lib)
```

## 二、调试技巧

```cmake
# 打印变量值
message(STATUS "CMAKE_CXX_COMPILER: ${CMAKE_CXX_COMPILER}")
message(STATUS "CMAKE_BUILD_TYPE: ${CMAKE_BUILD_TYPE}")

# 打印所有变量
get_cmake_property(_variableNames VARIABLES)
foreach(_variableName ${_variableNames})
    message(STATUS "${_variableName}=${${_variableName}}")
endforeach()
```

## 三、最佳实践

1. 使用现代 CMake（3.10+）
2. 使用 target_* 命令，避免全局命令
3. 明确 PUBLIC/PRIVATE/INTERFACE
4. 使用 find_package 查找依赖
5. 设置 C++ 标准
6. 使用生成器表达式处理平台差异
7. 分离源码和构建目录（out-of-source build）

## 四、常见面试题

### Q1: 一个 C++ 项目从源码到可执行文件的完整过程？

```
1. 预处理：g++ -E main.cpp -o main.i（处理 #include、#define 等）
2. 编译：g++ -S main.i -o main.s（生成汇编代码）
3. 汇编：g++ -c main.s -o main.o（生成目标文件）
4. 链接：g++ main.o utils.o -o app（符号解析、重定位）
```

### Q2: 如何在 CMake 中添加编译宏定义？

```cmake
# 推荐：target 级别
target_compile_definitions(myapp PRIVATE DEBUG_MODE FEATURE_X=1)

# 不推荐：全局
add_definitions(-DDEBUG_MODE)
```

### Q3: cmake 命令行常用参数？

```bash
cmake -B build                          # 指定构建目录
cmake -S src -B build                   # 指定源码和构建目录
cmake -DCMAKE_BUILD_TYPE=Debug ..       # 设置构建类型
cmake -DENABLE_TEST=ON ..               # 设置选项
cmake -G "Ninja" ..                     # 使用 Ninja 生成器
cmake --build build                     # 构建项目
cmake --install build                   # 安装项目
cmake --build build --target clean      # 清理
```

### Q4: Makefile 中 make -n 的作用？

只打印将要执行的命令，不实际执行（dry run）。

[src: raw/ingested/2技术/cpp/C++构建完整手册-11.-常见问题与技巧.md]

## Related Pages
- [[C++语言特性]]
- [[C++第三方库手册]]
- [[C++高频面试问题]]
