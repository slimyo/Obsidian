### **最小化学习路线（共 3 阶段，总计 1-2 天）**

#### **阶段 1：全局定位 + 历史背景（30-40 分钟）**

**目标**：知道 FasterTransformer 在整个 LLM Infra 历史中的位置。

1. 阅读以下两篇官方文档（最重要）：
    - **FasterTransformer GitHub 主 README.md**（前半部分）
    - **docs/gpt_guide.md**（前 1/3 即可）
2. 快速浏览 NVIDIA 官方博客（推荐）：
    - “Accelerated Inference for Large Transformer Models Using NVIDIA FasterTransformer” （2022 年那篇）

**这一步的收获**：你会明白 FasterTransformer 是 **2020-2023 年 NVIDIA 官方最强的 Transformer 推理加速库**，后来被 **TensorRT-LLM** 取代，但其设计思想被 vLLM 大量继承。

---

#### **阶段 2：高层次代码结构与工业模板（核心阶段，3-4 小时）**

**目标**：只看目录和关键文件，建立“AI Infra 工业标准模板”的整体印象。

**推荐阅读顺序（只看，不编译）**：

1. **整体架构地图**（30 分钟）
    
    - src/fastertransformer/models/
    - src/fastertransformer/layers/
    - src/fastertransformer/kernels/
2. **最值得重点看的 6 个代表性文件**（每个文件只看 15-25 分钟，重点看注释和类结构）：


|文件路径（相对 FasterTransformer 根目录）|核心作用|看什么|
|---|---|---|
|models/multi_gpu_gpt/ParallelGpt.cc|GPT 模型整体组装入口|看 Model 类如何把 Layer 拼起来|
|layers/attention_layers/GptContextAttentionLayer.cc|Context Attention（最经典的 Attention 实现）|看 Fused Attention 的结构|
|layers/attention_layers/DecoderSelfAttentionLayer.cc|Decoder Self-Attention|看 KV Cache + Fused 的设计|
|kernels/decoder_masked_multihead_attention.cu|经典 Fused Attention Kernel|只看函数签名和注释|
|utils/allocator.h + utils/buffer.h|内存管理|看 Allocator 的设计哲学|
|tensorrt_plugin/ 目录|TensorRT 插件封装|看工程集成方式|



**阅读技巧**：只看 .cc / .h / .cu 文件的**类定义、注释、大纲**，不用看具体 CUDA Kernel 内部实现。

---

#### **阶段 3：总结与印象固化（30-40 分钟）**

**任务**：合上代码，写下或默念以下**7 个核心印象**（这是你学习 FasterTransformer 的真正目的）。

---

### **你应该从 FasterTransformer 中获得的 7 个关键印象**

1. **工业级 LLM 推理的核心哲学是“高度融合（Fused）”** 把 Attention + Bias + Residual + LayerNorm 全部融合成一个 Kernel，最大化 GPU 利用率。这是 vLLM 后来 PagedAttention 等优化的基础理念。
2. **分层 + 模块化设计是工业标准模板** FasterTransformer 把代码严格分为：models/（模型组装）→ layers/（层）→ kernels/（算子）→ utils/（基础设施）。vLLM、TensorRT-LLM 都沿用了这个结构。
3. **Tensor Parallelism + Pipeline Parallelism 是大规模推理的标配** 即使你不看具体实现，也要记住：工业界做大模型推理，一定会用到这两种并行方式。
4. **内存管理是 LLM Infra 的生命线** FasterTransformer 对 KV Cache、Allocator、Buffer 的精细管理，直接影响吞吐量和显存占用。vLLM 的 PagedAttention 就是在解决同一问题。
5. **Kernel 融合与量化是性能的两个主要来源** Fused Kernel + INT8/FP8 量化，是早期 FasterTransformer 比 Hugging Face 快几倍的核心原因。
6. **从 FasterTransformer 到 TensorRT-LLM 的演进路径** NVIDIA 后来把 FT 的精华全部吸收到 TensorRT-LLM 中，所以 FT 是“老祖宗”级别的模板。
7. **vLLM 是在 FT 基础上的“下一代进化”** vLLM 继承了 FT 的“融合 + 并行 + 内存高效”理念，但增加了 **PagedAttention + 连续批处理（Continuous Batching）**，解决了 FT 在动态请求场景下的痛点。

---

### **最终建议（最小化执行）**

- **今天**：完成阶段 1 + 阶段 2（4 小时以内）。
- **明天**：完成阶段 3，写下 7 个印象。
- **总耗时**：不超过 **1.5 天**。

学完后，你对 **AI Infra 的工业标准模板** 会有非常清晰的认知，后续学习 vLLM 时，会明显感觉“很多设计我都见过”，学习曲线会陡然变平。

---

**下一步行动**： 如果你确认要开始，我马上给你：

1. **阶段 2 的精确文件清单 + 每个文件重点看哪几行**（带注释提示）。
2. **vLLM 学习前置知识对照表**（告诉你 FT 中哪些概念会直接对应 vLLM 的哪些模块）。