# Makefile

> Makefile 是 C/C++ 项目中最经典的构建工具，通过定义目标、依赖和命令来自动化编译流程。

## 1. 基本语法

```makefile
# 基本格式
目标(target): 依赖(dependencies)
    命令(command)  # 必须使用 Tab 缩进
```

## 2. 变量定义与使用

```makefile
# 变量赋值方式
VAR = value      # 递归展开变量，使用时才展开
VAR := value     # 简单展开变量，定义时立即展开
VAR ?= value     # 条件赋值，仅当变量未定义时赋值
VAR += value     # 追加值到已有变量

# 定义变量
CXX = g++
CXXFLAGS = -Wall -std=c++17 -O2
LDFLAGS = -lpthread

# 源文件和目标文件
SRCS = main.cpp utils.cpp network.cpp
OBJS = $(SRCS:.cpp=.o)
TARGET = myapp
```

## 3. 自动变量

| 变量 | 含义 |
|------|------|
| `$@` | 目标文件名 |
| `$<` | 第一个依赖文件名 |
| `$^` | 所有依赖文件（去重） |
| `$+` | 所有依赖文件（不去重） |
| `$?` | 所有比目标新的依赖文件 |
| `$*` | 目标文件去掉后缀的部分 |

## 4. 基本示例

```makefile
# 编译单个文件
hello: hello.cpp
    g++ -o hello hello.cpp

# 多文件项目
$(TARGET): $(OBJS)
    $(CXX) $(CXXFLAGS) -o $@ $^ $(LDFLAGS)

# 模式规则：将 .cpp 编译为 .o
%.o: %.cpp
    $(CXX) $(CXXFLAGS) -c $< -o $@

clean:
    rm -f $(OBJS) $(TARGET)
```

## 5. 伪目标

```makefile
.PHONY: all clean install test

all: $(TARGET)

clean:
    rm -f $(OBJS) $(TARGET)

install: $(TARGET)
    cp $(TARGET) /usr/local/bin/

test: $(TARGET)
    ./$(TARGET) --test
```

## 6. 条件判断

```makefile
DEBUG ?= 0

ifeq ($(DEBUG), 1)
    CXXFLAGS += -g -DDEBUG
else
    CXXFLAGS += -O2 -DNDEBUG
endif

ifeq ($(CC),gcc)
    CFLAGS += -std=gnu11
else
    CFLAGS += -std=c++17
endif

ifdef DEBUG
    CFLAGS += -g -DDEBUG
endif
```

## 7. 函数使用

```makefile
# 通配符与替换
SRCS = $(wildcard src/*.cpp)
OBJS = $(patsubst %.cpp,%.o,$(SRCS))
# 或简写
OBJS = $(SOURCES:.cpp=.o)

# 目录操作
DIRS = $(sort $(dir $(SRCS)))
FILES = $(notdir $(SRCS))
BASE = $(basename $(FILES))

# 字符串操作
UPPER = $(shell echo $(VAR) | tr a-z A-Z)
```

## 8. 自动生成依赖

```makefile
# 自动生成头文件依赖
DEPS = $(OBJS:.o=.d)

%.o: %.cpp
    $(CXX) $(CXXFLAGS) -MMD -MP -c $< -o $@

-include $(DEPS)
```

## 9. 多目录 Makefile 示例

```makefile
# 项目结构: project/src/, project/include/, project/lib/

CXX = g++
CXXFLAGS = -Wall -Wextra -std=c++17
CXXFLAGS += -Iinclude
LDFLAGS = 

DEBUG ?= 0
ifeq ($(DEBUG), 1)
    CXXFLAGS += -g -O0 -DDEBUG
else
    CXXFLAGS += -O2 -DNDEBUG
endif

SRC_DIR = src
OBJ_DIR = obj
BIN_DIR = bin

SRCS = $(wildcard $(SRC_DIR)/*.cpp)
OBJS = $(patsubst $(SRC_DIR)/%.cpp,$(OBJ_DIR)/%.o,$(SRCS))
DEPS = $(OBJS:.o=.d)
TARGET = $(BIN_DIR)/myapp

.PHONY: all clean dirs

all: dirs $(TARGET)

dirs:
    @mkdir -p $(OBJ_DIR) $(BIN_DIR)

$(TARGET): $(OBJS)
    $(CXX) $(CXXFLAGS) -o $@ $^ $(LDFLAGS)

$(OBJ_DIR)/%.o: $(SRC_DIR)/%.cpp
    $(CXX) $(CXXFLAGS) -MMD -MP -c $< -o $@

-include $(DEPS)

clean:
    rm -rf $(OBJ_DIR) $(BIN_DIR)
```

[src: raw/ingested/2技术/cpp/C++构建完整手册-2.-Makefile.md]

## Related Pages
- [[C++构建常见问题与技巧]]
- [[C++语言特性]]
- [[C++高频面试问题]]
