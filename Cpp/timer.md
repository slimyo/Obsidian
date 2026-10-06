**C++ 中常用的计时器 / 时间 API 用法**（基于 C++11 及以后标准库）

### 1. 测量时间（最常用）—— `<chrono>`

```cpp
#include <iostream>
#include <chrono>
#include <thread>   // 如果需要 sleep

using namespace std::chrono;

int main() {
    // 推荐用 steady_clock（不受系统时间调整影响）
    auto start = steady_clock::now();

    // 你的代码...
    std::this_thread::sleep_for(milliseconds(150));  // 模拟耗时

    auto end = steady_clock::now();

    // 计算耗时
    auto duration = duration_cast<milliseconds>(end - start);
    std::cout << "耗时: " << duration.count() << " ms\n";

    // 也可以用 microseconds、nanoseconds、seconds 等
    auto us = duration_cast<microseconds>(end - start).count();
    std::cout << "耗时: " << us << " μs\n";
}
```

常用时钟：
- `steady_clock`：单调时钟，适合测量时间间隔（推荐）
- `high_resolution_clock`：精度最高（实现可能是 `steady_clock` 或 `system_clock`）
- `system_clock`：系统墙钟时间，可转成日历时间

### 2. 延时（Sleep）

```cpp
#include <thread>
#include <chrono>

using namespace std::chrono_literals;  // 支持 100ms、2s 这种字面量

std::this_thread::sleep_for(100ms);          // 休眠 100 毫秒
std::this_thread::sleep_for(2s);             // 休眠 2 秒
std::this_thread::sleep_until(steady_clock::now() + 500ms);
```

### 3. 简单定时器（单次 / 周期性）—— 用线程实现

C++ 标准库没有现成的“定时器回调”类，通常自己用线程 + `sleep` 实现：

```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <atomic>
#include <functional>

class Timer {
public:
    // 单次定时
    void setTimeout(std::function<void()> func, int delay_ms) {
        std::thread([func, delay_ms]() {
            std::this_thread::sleep_for(std::chrono::milliseconds(delay_ms));
            func();
        }).detach();
    }

    // 周期性定时
    void setInterval(std::function<void()> func, int interval_ms) {
        std::thread([this, func, interval_ms]() {
            while (running_) {
                std::this_thread::sleep_for(std::chrono::milliseconds(interval_ms));
                if (running_) func();
            }
        }).detach();
    }

    void stop() { running_ = false; }

private:
    std::atomic<bool> running_{true};
};

// 使用示例
int main() {
    Timer t;

    t.setTimeout([]() {
        std::cout << "1 秒后执行一次\n";
    }, 1000);

    t.setInterval([]() {
        std::cout << "每 500ms 执行一次\n";
    }, 500);

    std::this_thread::sleep_for(std::chrono::seconds(3));
    t.stop();
}
```

### 4. 更高精度 / 平台相关

| 需求              | 推荐方式                              |
|-------------------|---------------------------------------|
| 跨平台测量时间    | `std::chrono::steady_clock`           |
| 高精度计时        | `high_resolution_clock` + `nanoseconds` |
| Windows 高精度    | `QueryPerformanceCounter`             |
| Linux 高精度      | `clock_gettime(CLOCK_MONOTONIC)`      |
| 定时器回调（复杂）| 第三方库：Boost.Asio、libuv、Qt 的 QTimer 等 |

### 5. 实用小技巧

```cpp
// 把 duration 转成 double 秒
double seconds = duration<double>(end - start).count();

// C++14 字面量更方便
using namespace std::chrono_literals;
auto d = 1.5s + 200ms;
```

如果你需要特定场景（比如游戏循环帧率控制、异步定时器、高精度性能分析等），告诉我具体需求，我可以给你更针对性的代码。