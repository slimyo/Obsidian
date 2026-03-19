
Nsight 是 NVIDIA 提供的一套强大的开发工具套件，用于分析和调试 CUDA 应用程序。根据你的使用场景（分析性能或调试代码），主要分为 **Nsight Systems**（系统级性能分析）和 **Nsight Compute**（内核级性能分析）以及 **Nsight Graphics**（图形和计算调试）。

以下是使用 Nsight 的通用步骤和主要工具介绍：

### 1. 主要工具概览

|工具名称|主要用途|适用场景|
|---|---|---|
|**Nsight Systems (nsys)**​|**系统级性能分析**​|分析整个应用的 CPU/GPU 时间线，找出瓶颈（如数据传输、内核启动延迟、同步问题）。|
|**Nsight Compute (ncu)**​|**内核级性能分析**​|深入分析单个 CUDA Kernel 的性能，查看占用率、内存吞吐量、指令统计等。|
|**Nsight Graphics**​|**图形与计算调试**​|调试图形渲染管线（如 Vulkan/DirectX）和 CUDA 代码，支持断点、变量检查等。|

### 2. 使用 Nsight Systems (命令行/CLI)

Nsight Systems 通常用于快速定位应用的整体瓶颈。

#### 步骤 1：收集数据

在终端运行你的程序，使用 `nsys profile`命令进行性能分析。

```bash
# 基本用法：运行程序并生成 .nsys-rep 报告文件
nsys profile -o my_report ./my_cuda_app

# 常用选项：
# -o <file> : 指定输出报告文件名
# --stats=true : 在终端显示统计摘要
# --trace=cuda,nvtx : 追踪 CUDA API 和 NVTX 标记
```

#### 步骤 2：查看报告

生成报告后，你可以使用 GUI 界面或命令行查看：

- **GUI 查看**：运行 `nsys-ui my_report.nsys-rep`打开可视化时间线。
    
- **CLI 查看**：运行 `nsys stats my_report.nsys-rep`查看文本摘要。
    

**分析重点**：在 GUI 中查看时间线，确认 GPU 是否被充分利用，是否存在大量的内存拷贝（HtoD/DtoH）阻塞了计算。

### 3. 使用 Nsight Compute (命令行/CLI)

Nsight Compute 用于深入分析 Kernel 的细节。

#### 步骤 1：收集内核数据

```bash
# 分析程序中的所有内核
ncu -o kernel_profile ./my_cuda_app

# 分析特定指标（如内存访问）
ncu --set detailed -k "MyKernelName" ./my_cuda_app
```

#### 步骤 2：查看报告

- **GUI 查看**：`ncu-ui kernel_profile.ncu-rep`
    
- **CLI 查看**：`ncu --import kernel_profile.ncu-rep`
    

**分析重点**：查看指标如 `Achieved Occupancy`（占用率）、`Memory Throughput`（内存吞吐量）以及是否有 `Divergent Branches`（分支分歧）。

### 4. 使用 Nsight 图形界面 (GUI)

如果你更喜欢图形化操作（例如在 Windows/Linux 桌面环境）：

1. **启动 IDE**：从开始菜单或终端启动 `nsight-systems`或 `nsight-compute`。
    
2. **新建会话**：
    
    - 点击 **File**​ -> **New Project**（或直接连接目标设备）。
        
    - 在 **Target**​ 选项卡中配置你的可执行文件路径和参数。
        
    
3. **开始分析**：点击 **Start**​ 按钮，工具会自动运行程序并收集数据。
    
4. **分析结果**：在时间线视图或指标视图中查看热点和瓶颈。
    

### 5. 结合 NVTX (NVIDIA Tools Extension) 进行标记

为了在时间线中更好地区分代码段，你可以在代码中添加 NVTX 标记：

```cpp
#include <nvtx3/nvToolsExt.h>

// 在代码中标记区域
nvtxRangePushA("My Compute Phase");
// ... 你的 CUDA 内核或计算代码
nvtxRangePop();
```

使用 `--trace=nvtx`运行 Nsight Systems 时，这些标记会显示在时间线上，帮助你对应代码和性能数据。

### 6. 常见使用场景示例

- **场景：程序跑得慢**
    
    - **先用 Nsight Systems**：看是 CPU 等 GPU 太久，还是 GPU 空闲太多。
        
    - **再用 Nsight Compute**：如果 GPU 忙但慢，深入看 Kernel 为什么慢（内存带宽受限？计算受限？）。
        
    
- **场景：调试崩溃或数值错误**
    
    - 使用 **Nsight Graphics**​ 或 **Nsight Compute**​ 的调试模式附加到进程，设置断点检查 GPU 内存和变量。
    

### 注意事项

- **环境**：确保你的系统已安装对应版本的 Nsight 工具（通常包含在 CUDA Toolkit 或单独下载）。
    
- **权限**：在 Linux 上可能需要调整 `/proc/sys/kernel/perf_event_paranoid`值或使用 `sudo`运行分析。
    
- **远程分析**：Nsight 支持连接到远程机器（如服务器）进行分析，需要在目标机器上启动 `nsight-sysem-cli`服务。
    

建议从 **Nsight Systems**​ 开始，先找到宏观瓶颈，再使用 **Nsight Compute**​ 进行微观优化。