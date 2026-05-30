# TBB 实际应用场景

> 本文展示 Intel TBB 在图像处理、数值计算和数据处理流水线中的实际应用示例。

See also: [[TBB最佳实践]], [[C++多线程与并发]], [[C++并发性能优化]]

## 6.1 图像处理

使用 `tbb::parallel_for` 和 `tbb::blocked_range2d` 对图像进行并行灰度化和反色处理。

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <vector>
#include <chrono>
#include <cmath>

// 简单的像素结构
struct Pixel {
    unsigned char r, g, b;
    
    // 灰度化处理
    Pixel grayscale() const {
        unsigned char gray = static_cast<unsigned char>(0.299 * r + 0.587 * g + 0.114 * b);
        return {gray, gray, gray};
    }
    
    // 反色处理
    Pixel invert() const {
        return {static_cast<unsigned char>(255 - r), 
                static_cast<unsigned char>(255 - g), 
                static_cast<unsigned char>(255 - b)};
    }
};

int main() {
    std::cout << "=== 并行图像处理示例 ===" << std::endl;
    
    const int WIDTH = 1920;
    const int HEIGHT = 1080;
    const int TOTAL_PIXELS = WIDTH * HEIGHT;
    
    // 创建模拟图像数据
    std::vector<Pixel> image(TOTAL_PIXELS);
    for (int i = 0; i < TOTAL_PIXELS; ++i) {
        image[i] = {
            static_cast<unsigned char>(i % 256),
            static_cast<unsigned char>((i * 2) % 256),
            static_cast<unsigned char>((i * 3) % 256)
        };
    }
    
    std::cout << "图像大小: " << WIDTH << "x" << HEIGHT << " = " 
              << TOTAL_PIXELS << " 像素" << std::endl;
    
    // 串行处理
    std::vector<Pixel> serial_result = image;
    auto start1 = std::chrono::high_resolution_clock::now();
    
    for (int y = 0; y < HEIGHT; ++y) {
        for (int x = 0; x < WIDTH; ++x) {
            int idx = y * WIDTH + x;
            serial_result[idx] = serial_result[idx].grayscale().invert();
        }
    }
    
    auto end1 = std::chrono::high_resolution_clock::now();
    auto time1 = std::chrono::duration_cast<std::chrono::milliseconds>(end1 - start1);
    
    // 并行处理（按行并行）
    std::vector<Pixel> parallel_result = image;
    auto start2 = std::chrono::high_resolution_clock::now();
    
    tbb::parallel_for(0, HEIGHT, [&](int y) {
        for (int x = 0; x < WIDTH; ++x) {
            int idx = y * WIDTH + x;
            parallel_result[idx] = parallel_result[idx].grayscale().invert();
        }
    });
    
    auto end2 = std::chrono::high_resolution_clock::now();
    auto time2 = std::chrono::duration_cast<std::chrono::milliseconds>(end2 - start2);
    
    // 并行处理（2D blocked_range）
    std::vector<Pixel> parallel_2d_result = image;
    auto start3 = std::chrono::high_resolution_clock::now();
    
    tbb::parallel_for(
        tbb::blocked_range2d<int>(0, HEIGHT, 0, WIDTH),
        [&](const tbb::blocked_range2d<int>& r) {
            for (int y = r.rows().begin(); y < r.rows().end(); ++y) {
                for (int x = r.cols().begin(); x < r.cols().end(); ++x) {
                    int idx = y * WIDTH + x;
                    parallel_2d_result[idx] = parallel_2d_result[idx].grayscale().invert();
                }
            }
        }
    );
    
    auto end3 = std::chrono::high_resolution_clock::now();
    auto time3 = std::chrono::duration_cast<std::chrono::milliseconds>(end3 - start3);
    
    // 验证结果
    bool results_match = true;
    for (int i = 0; i < 100; ++i) {
        if (serial_result[i].r != parallel_result[i].r) {
            results_match = false;
            break;
        }
    }
    
    std::cout << "\n性能对比:" << std::endl;
    std::cout << "  串行处理: " << time1.count() << " ms" << std::endl;
    std::cout << "  并行处理(按行): " << time2.count() << " ms" << std::endl;
    std::cout << "  并行处理(2D): " << time3.count() << " ms" << std::endl;
    std::cout << "  加速比: " << (double)time1.count() / time2.count() << "x" << std::endl;
    std::cout << "  结果验证: " << (results_match ? "通过" : "失败") << std::endl;
    
    // 显示处理前后的样本像素
    std::cout << "\n样本像素 (索引0):" << std::endl;
    std::cout << "  处理前: RGB(" << (int)image[0].r << ", " 
              << (int)image[0].g << ", " << (int)image[0].b << ")" << std::endl;
    std::cout << "  处理后: RGB(" << (int)parallel_result[0].r << ", "
              << (int)parallel_result[0].g << ", " << (int)parallel_result[0].b << ")" << std::endl;
    
    return 0;
}

/* 输出示例：
=== 并行图像处理示例 ===
图像大小: 1920x1080 = 2073600 像素

性能对比:
  串行处理: 45 ms
  并行处理(按行): 12 ms
  并行处理(2D): 11 ms
  加速比: 3.75x
  结果验证: 通过

样本像素 (索引0):
  处理前: RGB(0, 0, 0)
  处理后: RGB(255, 255, 255)
*/
```

## 6.2 数值计算

使用 `tbb::parallel_for` 实现并行矩阵乘法。

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <vector>
#include <chrono>
#include <iomanip>

// 简单的矩阵类
class Matrix {
public:
    std::vector<double> data;
    int rows_, cols_;
    
    Matrix(int rows, int cols) : rows_(rows), cols_(cols), data(rows * cols, 0.0) {}
    
    double& operator()(int i, int j) { return data[i * cols_ + j]; }
    double operator()(int i, int j) const { return data[i * cols_ + j]; }
    
    int rows() const { return rows_; }
    int cols() const { return cols_; }
    
    void print(const std::string& name, int max_rows = 4, int max_cols = 4) const {
        std::cout << name << " (" << rows_ << "x" << cols_ << "):" << std::endl;
        for (int i = 0; i < std::min(rows_, max_rows); ++i) {
            std::cout << "  [";
            for (int j = 0; j < std::min(cols_, max_cols); ++j) {
                std::cout << std::setw(8) << std::fixed << std::setprecision(1) << (*this)(i, j);
            }
            if (cols_ > max_cols) std::cout << " ...";
            std::cout << " ]" << std::endl;
        }
        if (rows_ > max_rows) std::cout << "  ..." << std::endl;
    }
};

// 串行矩阵乘法
Matrix serial_multiply(const Matrix& A, const Matrix& B) {
    Matrix C(A.rows(), B.cols());
    
    for (int i = 0; i < A.rows(); ++i) {
        for (int j = 0; j < B.cols(); ++j) {
            double sum = 0;
            for (int k = 0; k < A.cols(); ++k) {
                sum += A(i, k) * B(k, j);
            }
            C(i, j) = sum;
        }
    }
    
    return C;
}

// 并行矩阵乘法
Matrix parallel_multiply(const Matrix& A, const Matrix& B) {
    Matrix C(A.rows(), B.cols());
    
    tbb::parallel_for(0, A.rows(), [&](int i) {
        // 并发正确性要点：
        // 1) i 被 TBB 按区间划分到不同任务/worker；同一时刻不同任务处理不同的行 i。
        // 2) B 在该函数签名里是 const，只会被读取（A(i,k) * B(k,j)）；并行“读共享”是安全的。
        // 3) 每个任务只写 C(i, j)（即 C 的某一整行），不同 i 对应的写入区域互不重叠，因此不会发生数据竞争。
        //
        // 性能要点（不影响并发正确性）：
        // - 虽然我们“按行”并行，但内层对 B 的访问是 B(k, j)（按列方向跨 k 遍历），这更不利于缓存局部性；
        //   它通常表现为更慢，而不是错误结果。
        for (int j = 0; j < B.cols(); ++j) {
            double sum = 0;
            for (int k = 0; k < A.cols(); ++k) {//a的列数跟b的行数相同
                sum += A(i, k) * B(k, j);
            }
            C(i, j) = sum;
        }
    });
    
    return C;
}

int main() {
    std::cout << "=== 并行矩阵乘法示例 ===" << std::endl;
    
    const int M = 500;  // A的行数
    const int K = 400;  // A的列数 = B的行数
    const int N = 300;  // B的列数
    
    // 初始化矩阵
    Matrix A(M, K), B(K, N);
    
    for (int i = 0; i < M; ++i)
        for (int j = 0; j < K; ++j)
            A(i, j) = (i + j) % 10;
    
    for (int i = 0; i < K; ++i)
        for (int j = 0; j < N; ++j)
            B(i, j) = (i * j) % 10;
    
    std::cout << "\n矩阵大小: A(" << M << "x" << K << ") * B(" << K << "x" << N 
              << ") = C(" << M << "x" << N << ")" << std::endl;
    
    // 串行乘法
    auto start1 = std::chrono::high_resolution_clock::now();
    Matrix C1 = serial_multiply(A, B);
    auto end1 = std::chrono::high_resolution_clock::now();
    auto time1 = std::chrono::duration_cast<std::chrono::milliseconds>(end1 - start1);
    
    // 并行乘法
    auto start2 = std::chrono::high_resolution_clock::now();
    Matrix C2 = parallel_multiply(A, B);
    auto end2 = std::chrono::high_resolution_clock::now();
    auto time2 = std::chrono::duration_cast<std::chrono::milliseconds>(end2 - start2);
    
    // 验证结果
    bool match = true;
    for (int i = 0; i < M && match; ++i) {
        for (int j = 0; j < N && match; ++j) {
            if (std::abs(C1(i, j) - C2(i, j)) > 1e-9) {
                match = false;
            }
        }
    }
    
    std::cout << "\n性能对比:" << std::endl;
    std::cout << "  串行乘法: " << time1.count() << " ms" << std::endl;
    std::cout << "  并行乘法: " << time2.count() << " ms" << std::endl;
    std::cout << "  加速比: " << std::fixed << std::setprecision(2) 
              << (double)time1.count() / time2.count() << "x" << std::endl;
    std::cout << "  结果验证: " << (match ? "通过" : "失败") << std::endl;
    
    // 显示小矩阵示例
    std::cout << "\n=== 小矩阵示例演示 ===" << std::endl;
    Matrix smallA(3, 2), smallB(2, 3);
    smallA(0,0)=1; smallA(0,1)=2;
    smallA(1,0)=3; smallA(1,1)=4;
    smallA(2,0)=5; smallA(2,1)=6;
    
    smallB(0,0)=7; smallB(0,1)=8; smallB(0,2)=9;
    smallB(1,0)=10; smallB(1,1)=11; smallB(1,2)=12;
    
    Matrix smallC = parallel_multiply(smallA, smallB);
    
    smallA.print("A");
    smallB.print("B");
    smallC.print("C = A * B");
    
    return 0;
}

/* 输出示例：
=== 并行矩阵乘法示例 ===

矩阵大小: A(500x400) * B(400x300) = C(500x300)

性能对比:
  串行乘法: 245 ms
  并行乘法: 68 ms
  加速比: 3.60x
  结果验证: 通过

=== 小矩阵示例演示 ===
A (3x2):
  [     1.0     2.0 ]
  [     3.0     4.0 ]
  [     5.0     6.0 ]
B (2x3):
  [     7.0     8.0     9.0 ]
  [    10.0    11.0    12.0 ]
C = A * B (3x3):
  [    27.0    30.0    33.0 ]
  [    61.0    68.0    75.0 ]
  [    95.0   106.0   117.0 ]
*/
```

## 6.3 数据处理流水线

使用 TBB Flow Graph 实现数据处理流水线，包含读取、处理和写入三个阶段。

```cpp
#include <tbb/tbb.h>
#include <iostream>
#include <string>
#include <sstream>
#include <thread>
#include <chrono>
#include <mutex>
#include <atomic>

std::mutex cout_mutex;

// 模拟数据记录
struct Record {
    int id;
    std::string raw_data;
};

// 处理后的数据
struct ProcessedRecord {
    int id;
    std::string transformed_data;
    int word_count;
};

int main() {
    std::cout << "=== 数据处理流水线示例 ===" << std::endl;
    
    using namespace tbb::flow;
    graph g;
    
    std::atomic<int> records_read{0};
    std::atomic<int> records_processed{0};
    std::atomic<int> records_written{0};
    
    // 模拟数据源
    std::vector<Record> input_data = {
        {1, "hello world"},
        {2, "tbb is awesome"},
        {3, "parallel programming"},
        {4, "flow graph example"},
        {5, "data pipeline demo"},
        {6, "concurrent processing"},
        {7, "high performance"},
        {8, "multi threaded"}
    };
    
    std::atomic<size_t> current_index{0};
    
    // 阶段1: 数据源节点 - 读取数据
    source_node<Record> reader(g, [&](Record& record) -> bool {
        size_t idx = current_index.fetch_add(1);
        if (idx >= input_data.size()) {
            return false;  // 没有更多数据
        }
        record = input_data[idx];
        records_read++;
        
        // 模拟读取延迟
        std::this_thread::sleep_for(std::chrono::milliseconds(20));
        
        {
            std::lock_guard<std::mutex> lock(cout_mutex);
            std::cout << "[读取] ID=" << record.id << ": \"" << record.raw_data << "\"" << std::endl;
        }
        return true;
    }, false);  // false表示不自动激活
    
    // 阶段2: 处理节点 - 转换数据（可并行）
    function_node<Record, ProcessedRecord> processor(g, unlimited,
        [&](const Record& record) -> ProcessedRecord {
            // 模拟处理延迟
            std::this_thread::sleep_for(std::chrono::milliseconds(50));
            
            // 转换为大写并计算单词数
            std::string upper;
            int word_count = 1;
            for (char c : record.raw_data) {
                if (c == ' ') word_count++;
                upper += std::toupper(c);
            }
            
            records_processed++;
            
            {
                std::lock_guard<std::mutex> lock(cout_mutex);
                std::cout << "[处理] ID=" << record.id << ": \"" << upper 
                          << "\" (词数:" << word_count << ")" << std::endl;
            }
            
            return {record.id, upper, word_count};
        });
    
    // 阶段3: 写入节点 - 保存结果（串行保证顺序）
    std::vector<ProcessedRecord> output_data;
    function_node<ProcessedRecord> writer(g, serial,
        [&](const ProcessedRecord& record) {
            // 模拟写入延迟
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
            
            output_data.push_back(record);
            records_written++;
            
            {
                std::lock_guard<std::mutex> lock(cout_mutex);
                std::cout << "[写入] ID=" << record.id << " 完成" << std::endl;
            }
        });
    
    // 连接流水线
    make_edge(reader, processor);
    make_edge(processor, writer);
    
    std::cout << "\n开始处理 " << input_data.size() << " 条记录...\n" << std::endl;
    
    auto start = std::chrono::high_resolution_clock::now();
    
    // 激活数据源并等待完成
    reader.activate();
    g.wait_for_all();
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "\n========== 处理完成 ==========" << std::endl;
    std::cout << "统计信息:" << std::endl;
    std::cout << "  读取: " << records_read << " 条" << std::endl;
    std::cout << "  处理: " << records_processed << " 条" << std::endl;
    std::cout << "  写入: " << records_written << " 条" << std::endl;
    std::cout << "  总耗时: " << duration.count() << " ms" << std::endl;
    
    // 如果串行执行: 8*(20+50+10) = 640ms
    // 流水线并行可以显著减少时间
    std::cout << "  串行预估: " << input_data.size() * (20 + 50 + 10) << " ms" << std::endl;
    
    std::cout << "\n输出结果:" << std::endl;
    for (const auto& r : output_data) {
        std::cout << "  {id=" << r.id << ", data=\\"" << r.transformed_data 
                  << "\", words=" << r.word_count << "}" << std::endl;
    }
    
    return 0;
}

/* 输出示例：
=== 数据处理流水线示例 ===

开始处理 8 条记录...

[读取] ID=1: "hello world"
[读取] ID=2: "tbb is awesome"
[处理] ID=1: "HELLO WORLD" (词数:2)
[读取] ID=3: "parallel programming"
[处理] ID=2: "TBB IS AWESOME" (词数:3)
[写入] ID=1 完成
[读取] ID=4: "flow graph example"
[处理] ID=3: "PARALLEL PROGRAMMING" (词数:2)
[写入] ID=2 完成
...

========== 处理完成 ==========
统计信息:
  读取: 8 条
  处理: 8 条
  写入: 8 条
  总耗时: 285 ms
  串行预估: 640 ms

输出结果:
  {id=1, data="HELLO WORLD", words=2}
  {id=2, data="TBB IS AWESOME", words=3}
  {id=3, data="PARALLEL PROGRAMMING", words=2}
  {id=4, data="FLOW GRAPH EXAMPLE", words=3}
  {id=5, data="DATA PIPELINE DEMO", words=3}
  {id=6, data="CONCURRENT PROCESSING", words=2}
  {id=7, data="HIGH PERFORMANCE", words=2}
  {id=8, data="MULTI THREADED", words=2}
*/
```

[src: raw/ingested/2技术/cpp/并行库-C++_tbb库分析-六、实际应用场景-六、实际应用场景.md]