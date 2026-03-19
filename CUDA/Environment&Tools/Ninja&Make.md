Make 和 Ninja 都是用于自动化构建软件项目的工具，它们通过读取构建规则文件（如 Makefile 或 build.ninja）来管理编译、链接等任务，但两者在设计和性能上有所不同。

### 1. 简述

- **Make**：最经典、历史最悠久的构建工具之一，通常与 Unix/Linux 系统紧密集成。它使用 Makefile 文件来描述源文件与目标文件之间的依赖关系。
    
- **Ninja**：一个专注于**速度**的构建系统，由 Google 开发。它旨在作为其他构建系统（如 CMake、GYP）的后端，生成 build.ninja 文件，然后由 Ninja 执行。
    

### 2. 主要区别

|维度|Make|Ninja|
|---|---|---|
|**设计目标**​|通用、功能丰富，支持复杂的逻辑和 shell 命令。|**极致速度**，启动开销低，增量构建快。|
|**配置文件**​|**Makefile**：语法相对复杂，支持条件判断、函数等编程特性。|**build.ninja**：语法极简，通常由更高级的工具（如 CMake）生成，不适合手写。|
|**依赖检查**​|通常基于文件时间戳（timestamp），有时不够精确。|通常更严格，支持校验和（checksum）检查，避免不必要的重复构建。|
|**并行构建**​|支持（`make -jN`），但调度开销相对较大。|**高度优化**的并行构建，能更高效地利用多核 CPU。|
|**使用场景**​|手动管理的小型项目，或直接调用编译器。|大型项目（如 Chromium、Android），通常作为 CMake 或 Meson 的后端。|

### 3. 核心差异总结

- **Make 更像“脚本”**：你可以直接在 Makefile 里写复杂的逻辑，适合直接控制构建过程。
    
- **Ninja 更像“引擎”**：它本身不处理复杂的配置逻辑，只负责以最快的速度执行别人（如 CMake）给它生成的构建计划。对于大型项目，Ninja 的构建速度通常显著快于 Make。


### 常用Ninja命令

- 构建：
```
# 1. 标准 Debug 构建（用于开发调试）
cmake -G Ninja -B build/debug -DCMAKE_BUILD_TYPE=Debug

# 2. 标准 Release 构建（用于性能测试和发布）
cmake -G Ninja -B build/release -DCMAKE_BUILD_TYPE=Release

# 3. 带有调试信息的 Release 构建（用于线上问题分析）
cmake -G Ninja -B build/relwithdebinfo -DCMAKE_BUILD_TYPE=RelWithDebInfo

# 4. 开启 Clang 高级工具链（内存检测、代码覆盖率等）
# 4.1 地址消毒器 (AddressSanitizer)
cmake -G Ninja -B build/asan -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address -fno-omit-frame-pointer"

# 4.2 代码覆盖率 (Code Coverage)
cmake -G Ninja -B build/coverage -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="--coverage"

# 5. 启用更严格的编译检查
cmake -G Ninja -B build/strict \
  -DCMAKE_CXX_FLAGS="-Wall -Wextra -Werror -Wpedantic -Wconversion"

# 配置完成后，进入相应目录使用 ninja 构建
cd build/debug && ninja
# 或者直接
ninja -C build/debug
```

- 清除
```
# 对于 Make 生成器
cd build && make clean
# 或从外部调用
make -C build clean

# 对于 Ninja 生成器
cd build && ninja clean
# 或从外部调用
ninja -C build clean

# 使用 CMake 的统一构建命令
cmake --build build --target clean
```
