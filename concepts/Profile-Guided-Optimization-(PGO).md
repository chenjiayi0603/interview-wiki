# Profile-Guided Optimization (PGO)

**原理**：根据实际运行数据优化代码，编译器根据真实执行路径优化分支预测、函数内联和代码布局。

```bash
# 1. 编译时添加 -fprofile-generate
g++ -O3 -fprofile-generate -o program program.cpp

# 2. 运行程序生成 profile 数据
./program < typical_input

# 3. 使用 profile 数据重新编译
g++ -O3 -fprofile-use -o program_optimized program.cpp
```

**完整示例：PGO 优化实战**：

```cpp
// pgo_demo.cpp - PGO 优化演示程序
#include <iostream>
#include <vector>
#include <string>
#include <chrono>
#include <random>
#include <algorithm>
#include <unordered_map>

// ==================== 模拟真实业务场景 ====================

// 订单类型
enum class OrderType {
    MARKET,     // 市价单（最常见，约70%）
    LIMIT,      // 限价单（约25%）
    STOP,       // 止损单（约4%）
    STOP_LIMIT  // 止损限价单（约1%）
};

// 订单结构
struct Order {
    int order_id;
    OrderType type;
    double price;
    int quantity;
    std::string symbol;
};

// 处理不同类型订单的函数
class OrderProcessor {
public:
    // 分支密集型函数 - PGO 可以优化分支预测
    double process_order(const Order& order) {
        double result = 0.0;
        
        switch (order.type) {
            case OrderType::MARKET:
                // 最常执行的路径
                result = process_market_order(order);
                break;
            case OrderType::LIMIT:
                result = process_limit_order(order);
                break;
            case OrderType::STOP:
                result = process_stop_order(order);
                break;
            case OrderType::STOP_LIMIT:
                result = process_stop_limit_order(order);
                break;
        }
        
        // 验证处理结果
        if (result < 0) {
            // 很少执行的错误处理路径
            handle_error(order);
            return -1;
        }
        
        return result;
    }
    
    // 批量处理函数
    double process_batch(const std::vector<Order>& orders) {
        double total = 0.0;
        for (const auto& order : orders) {
            total += process_order(order);
        }
        return total;
    }
    
private:
    // 市价单处理（最常调用，PGO 会优先优化）
    double process_market_order(const Order& order) {
        // 模拟复杂计算
        double price = order.price * 1.001;  // 加滑点
        if (order.quantity > 1000) {
            price *= 1.0005;  // 大单额外滑点
        }
        return price * order.quantity;
    }
    
    // 限价单处理
    double process_limit_order(const Order& order) {
        double price = order.price;
        // 模拟价格检查
        if (price <= 0) return -1;
        return price * order.quantity;
    }
    
    // 止损单处理
    double process_stop_order(const Order& order) {
        double trigger_price = order.price * 0.95;
        return trigger_price * order.quantity;
    }
    
    // 止损限价单处理（最少调用）
    double process_stop_limit_order(const Order& order) {
        double trigger_price = order.price * 0.95;
        double limit_price = order.price * 0.94;
        return std::min(trigger_price, limit_price) * order.quantity;
    }
    
    void handle_error(const Order& order) {
        std::cerr << "Error processing order: " << order.order_id << std::endl;
    }
};

// ==================== 字符串处理场景 ====================

class StringProcessor {
public:
    // 模式匹配 - 不同模式出现频率不同
    int categorize_message(const std::string& msg) {
        // 假设 "TRADE" 开头的消息最多（60%）
        if (msg.substr(0, 5) == "TRADE") {
            return process_trade_message(msg);
        }
        // "QUOTE" 开头的消息次多（30%）
        else if (msg.substr(0, 5) == "QUOTE") {
            return process_quote_message(msg);
        }
        // "ADMIN" 开头的消息最少（10%）
        else if (msg.substr(0, 5) == "ADMIN") {
            return process_admin_message(msg);
        }
        return 0;
    }
    
private:
    int process_trade_message(const std::string& msg) {
        return static_cast<int>(msg.length() * 2);
    }
    
    int process_quote_message(const std::string& msg) {
        return static_cast<int>(msg.length() * 3);
    }
    
    int process_admin_message(const std::string& msg) {
        return static_cast<int>(msg.length() * 5);
    }
};

// ==================== 生成测试数据 ====================

std::vector<Order> generate_orders(int count) {
    std::vector<Order> orders;
    orders.reserve(count);
    
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<> type_dist(1, 100);
    std::uniform_real_distribution<> price_dist(10.0, 1000.0);
    std::uniform_int_distribution<> qty_dist(100, 10000);
    
    for (int i = 0; i < count; ++i) {
        Order order;
        order.order_id = i;
        order.price = price_dist(gen);
        order.quantity = qty_dist(gen);
        order.symbol = "STOCK" + std::to_string(i % 100);
        
        // 模拟真实分布：MARKET 70%, LIMIT 25%, STOP 4%, STOP_LIMIT 1%
        int r = type_dist(gen);
        if (r <= 70) {
            order.type = OrderType::MARKET;
        } else if (r <= 95) {
            order.type = OrderType::LIMIT;
        } else if (r <= 99) {
            order.type = OrderType::STOP;
        } else {
            order.type = OrderType::STOP_LIMIT;
        }
        
        orders.push_back(order);
    }
    
    return orders;
}

std::vector<std::string> generate_messages(int count) {
    std::vector<std::string> messages;
    messages.reserve(count);
    
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<> type_dist(1, 100);
    
    for (int i = 0; i < count; ++i) {
        int r = type_dist(gen);
        if (r <= 60) {
            messages.push_back("TRADE:BUY:AAPL:100:150.25");
        } else if (r <= 90) {
            messages.push_back("QUOTE:AAPL:150.20:150.30");
        } else {
            messages.push_back("ADMIN:HEARTBEAT:OK");
        }
    }
    
    return messages;
}

// ==================== 性能测试 ====================

int main() {
    const int ORDER_COUNT = 1000000;
    const int MESSAGE_COUNT = 1000000;
    const int ITERATIONS = 10;
    
    std::cout << "=== PGO 优化测试程序 ==" << std::endl;
    std::cout << "订单数量: " << ORDER_COUNT << std::endl;
    std::cout << "消息数量: " << MESSAGE_COUNT << std::endl;
    std::cout << "迭代次数: " << ITERATIONS << std::endl;
    std::cout << std::endl;
    
    // 生成测试数据
    auto orders = generate_orders(ORDER_COUNT);
    auto messages = generate_messages(MESSAGE_COUNT);
    
    OrderProcessor order_processor;
    StringProcessor string_processor;
    
    // 测试订单处理性能
    auto start = std::chrono::high_resolution_clock::now();
    double order_total = 0;
    for (int i = 0; i < ITERATIONS; ++i) {
        order_total += order_processor.process_batch(orders);
    }
    auto end = std::chrono::high_resolution_clock::now();
    auto order_duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "订单处理耗时: " << order_duration.count() << " 毫秒" << std::endl;
    std::cout << "订单处理总额: " << order_total << std::endl;
    
    // 测试消息处理性能
    start = std::chrono::high_resolution_clock::now();
    long long msg_total = 0;
    for (int i = 0; i < ITERATIONS; ++i) {
        for (const auto& msg : messages) {
            msg_total += string_processor.categorize_message(msg);
        }
    }
    end = std::chrono::high_resolution_clock::now();
    auto msg_duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "消息处理耗时: " << msg_duration.count() << " 毫秒" << std::endl;
    std::cout << "消息处理总计: " << msg_total << std::endl;
    
    std::cout << "\n总执行时间: " << (order_duration.count() + msg_duration.count()) << " 毫秒" << std::endl;
    
    return 0;
}
```

**PGO 是啥？**

PGO（Profile-Guided Optimization，性能分析驱动优化）是一种现代编译器优化技术。其原理是：先让程序在真实或接近真实的负载下运行一次，收集热点路径和分支概率等行为数据，然后让编译器利用这些行为信息，对热点代码、分支跳转等做出更智能的优化，从而进一步提升性能。

简言之：**PGO 就是让编译器“看你程序实际怎么跑”，然后按真实场景针对性地做优化，而不是靠静态猜测**。

**典型应用优势：**
- 优化分支预测和内存局部性，更适合你的实际用例
- 自动调整内联、循环展开、函数布局等，提高主路径性能
- 在金融系统、Web 服务器等高负载场景下普遍能提升 5%~25% 实际性能

**完整 PGO 编译流程**：

```bash
# ==================== 第一步：生成插桩版本 ====================
# 编译生成 profile 数据的版本
g++ -O3 -march=native -fprofile-generate=./pgo_data -o pgo_demo_instrumented pgo_demo.cpp

# 确保 pgo_data 目录存在
mkdir -p pgo_data

# ==================== 第二步：运行并收集 profile ====================
# 运行插桩程序多次，模拟真实工作负载
echo "运行第1次..."
./pgo_demo_instrumented
echo "运行第2次..."
./pgo_demo_instrumented
echo "运行第3次..."
./pgo_demo_instrumented

# 查看生成的 profile 文件
ls -la pgo_data/
# 会看到 *.gcda 文件

# ==================== 第三步：使用 profile 重新编译 ====================
g++ -O3 -march=native -fprofile-use=./pgo_data -o pgo_demo_optimized pgo_demo.cpp

# ==================== 第四步：对比测试 ====================
# 编译无 PGO 版本作为对照
g++ -O3 -march=native -o pgo_demo_normal pgo_demo.cpp

echo "=== 无 PGO 版本 =="
./pgo_demo_normal

echo ""
echo "=== 有 PGO 版本 =="
./pgo_demo_optimized

# ==================== 典型输出对比 ====================
# 无 PGO 版本:
# 订单处理耗时: 1250 毫秒
# 消息处理耗时: 980 毫秒
# 总执行时间: 2230 毫秒

# 有 PGO 版本:
# 订单处理耗时: 1050 毫秒 (提升16%)
# 消息处理耗时: 820 毫秒  (提升16%)
# 总执行时间: 1870 毫秒  (总体提升16%)
```

**PGO 脚本自动化**：

```bash
#!/bin/bash
# pgo_build.sh - PGO 自动化构建脚本

SOURCE_FILE="pgo_demo.cpp"
PGO_DIR="./pgo_data"
OUTPUT="pgo_demo_optimized"
CXXFLAGS="-O3 -march=native -std=c++17"

# 清理旧数据
rm -rf ${PGO_DIR}
mkdir -p ${PGO_DIR}

echo "Step 1: 编译插桩版本..."
g++ ${CXXFLAGS} -fprofile-generate=${PGO_DIR} -o ${OUTPUT}_instrumented ${SOURCE_FILE}

echo "Step 2: 收集 profile 数据..."
for i in {1..5}; do
    echo "  运行第 $i 次..."
    ./${OUTPUT}_instrumented > /dev/null
done

echo "Step 3: 使用 profile 重新编译..."
g++ ${CXXFLAGS} -fprofile-use=${PGO_DIR} -o ${OUTPUT} ${SOURCE_FILE}

echo "Step 4: 清理临时文件..."
rm -f ${OUTPUT}_instrumented

echo "完成！优化后的可执行文件: ${OUTPUT}"
echo "文件大小: $(ls -lh ${OUTPUT} | awk '{print $5}')"
```

**Clang PGO 流程（略有不同）**：

```bash
# Clang 使用不同的参数
# 生成插桩版本
clang++ -O3 -fprofile-instr-generate=code.profraw -o program_instrumented program.cpp

# 运行并生成数据
./program_instrumented

# 合并 profile 数据
llvm-profdata merge -output=code.profdata code.profraw

# 使用 profile 编译
clang++ -O3 -fprofile-instr-use=code.profdata -o program_optimized program.cpp
```

[src: raw/ingested/2技术/性能优化/编译优化-编译优化-#-5.3-Profile-Guided-Optimization-(PGO).md]