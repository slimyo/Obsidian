[Visualization and Debugging | tinygrad/tinygrad | DeepWiki](https://deepwiki.com/tinygrad/tinygrad/3.3-visualization-and-debugging)
本页记录了 tinygrad 的可视化与调试基础设施，它使开发者能够检查 UOp 计算图的变换、调试编译过程，以及分析内核执行性能。该系统由三个主要组件构成：

- **基于网络的可视化服务器 (tinygrad/viz/serve.py)**​ - 提供交互式 UOp 图和时间线分析的 HTTP 服务器。
    
- **时间线分析器**​ - 基于 Canvas 的执行时间线，包含内存追踪和硬件特定的计数器。
    
- **设备特定的调试工具**​ - 用于获取精确到时钟周期指令追踪和汇编分析的 AMD SQTT 集成。
    

通过在执行时设置环境变量 `VIZ=1`即可访问可视化系统，该系统会在 `http://127.0.0.1:8000`启动一个服务器。为了降低开销或进行离线分析，设置 `VIZ=-1`可将追踪数据保存到一个 pickle 文件中，供后续通过 CLI 工具使用。

---

**可视化服务器架构**

可视化服务器使用 `TCPServerWithReuse`来托管一个网页界面。`HTTPRequestHandler`处理对重写上下文、图数据和分析时间线的请求。

**关键服务器组件**

|组件|位置|用途|
|---|---|---|
|`TCPServerWithReuse`|`tinygrad/viz/serve.py:15-19`|允许地址重用的 HTTP 服务器。|
|`HTTPRequestHandler`|`tinygrad/viz/serve.py:21-40`|支持 SSE（服务器发送事件）的请求处理器。|
|`get_rewrites()`|`tinygrad/viz/serve.py:67-79`|列出 `RewriteTrace`中捕获的所有重写上下文和步骤。|
|`uop_to_json()`|`tinygrad/viz/serve.py:98-149`|将 UOp 对象序列化为前端可用的 JSON 图格式。|
|`sqtt_timeline()`|`tinygrad/viz/serve.py:310-338`|将 AMD SQTT 硬件数据包解码为时间线事件。|

---

**图形可视化与调试**

图形可视化以交互调试功能渲染 UOp 计算图。布局计算在 Web Worker (`tinygrad/viz/js/worker.js`) 中完成，使用了 dagre 库。

**UOp 序列化与排除逻辑**

`uop_to_json`函数执行多项转换以使大型图易于阅读：

- **节点排除**：常见节点如 `Ops.DEVICE`, `Ops.CONST`, 和 `Ops.UNIQUE`从主图中排除，并内联到其父节点的标签中，以减少混乱。
    
- **颜色编码**：节点根据其操作类型着色。例如，`LOAD`为 `#ffc0c0`，`STORE`为 `#87CEEB`，`ALU`操作为 `#ffffc0`。
    
- **参数格式化**：像 `Ops.SHRINK`或 `Ops.PAD`这样的移动操作，其参数会使用 `mask_to_str`格式化为形状或掩码。
    

**基于 Worker 的布局 (worker.js)**

- 布局在单独线程中计算，以保持用户界面的响应性。
    
- `layoutUOp`：计算 UOp 图中节点的位置，处理 `CALL`节点展开和 `SINK`节点可见性。它应用 `callSrcMask`以选择性地隐藏 `CALL`节点的输入缓冲区。
    
- `layoutCfg`：为汇编级调试，使用基本块和分支路径计算控制流图 (CFG) 的位置。
    

---

**AMD SQTT 与硬件性能分析**

Tinygrad 深度集成了 AMD 的 SQ Thread Trace (SQTT) 硬件。这使得开发者能够精确看到在特定时间、哪个 SIMD 单元上执行了哪些指令。

**SQTT 数据包解码**

硬件特定的数据包结构被处理以提取指令级计时信息。

- **指令映射**：来自硬件数据包的 PC（程序计数器）值与反汇编的 ELF 二进制文件相关联。
    
- **时间线生成**：`sqtt_timeline()`将原始的基于 nibble 的数据包转换为 VIZ 时间线所需的 `ProfileRangeEvent`对象，允许用户查看“WAVE”和“EXEC”行。
    

**ROCProfiler 解码器集成**

为了高保真度的指令追踪，tinygrad 可以与 `rocprof-trace-decoder`库交互。

|类|用途|
|---|---|
|`WaveExec`|表示一个单一的 wavefront 执行，包含一系列指令时序。|
|`InstExec`|表示单个指令的执行，包括停顿和持续时间。|
|`sqtt_timeline`|生成可视化指令时间线的核心函数。|

---

**性能分析器时间线与内存追踪**

性能分析器提供了一个基于 canvas 的时间线 (`tinygrad/viz/js/index.js`)，用于可视化执行过程和内存使用。

**时间线布局与渲染**

- **二进制编码**：服务器将时间线数据编码为二进制格式，以减少传输延迟。
    
- **内存多边形**：内存使用情况被可视化为多边形，显示随时间变化的内存分配偏移和大小，便于检测内存碎片或过度分配。
    
- **D3 缩放**：界面使用 `d3.zoom`实现无缝导航，从毫秒级的内核视图到纳秒级的指令追踪。
    

**事件类型**

- `ProfileRangeEvent`：表示具有开始和结束时间的任务（例如，内核执行）。
    
- `ProfilePointEvent`：表示瞬时事件（例如，内存分配或频率采样）。
    
- `ProfileProgramEvent`：将设备和名称与已编译的二进制文件关联，用于反汇编。
    

---

**命令行界面 (CLI)**

`extra/viz/cli.py`工具允许在没有网页浏览器的情况下检查追踪数据，这在 CI/CD 或远程调试中非常有用。

**CLI 使用模式**

- **性能分析检查**：`extra/viz/cli.py --profile`汇总内核执行时间并打印摘要表格。
    
- **重写检查**：不带标志的 `extra/viz/cli.py`会列出追踪文件中捕获的内核的变换步骤。
    
- **SQTT 分析**：CLI 可以使用 `--src`标志打印精确到时钟周期的指令日志，包含单元、操作码、持续时间和 PC 信息。