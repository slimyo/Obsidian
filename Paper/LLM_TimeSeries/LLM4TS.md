arXiv:2308.08469v6，2023/2025，Ching Chang,

- LLM4TS提出一个“两阶段微调”框架——微调策略
	1. Time-Series Alignment 时序对齐阶段，（理解时序特征）
	2. Forecasting Fine-Tuning，预测微调阶段
- 结合
	1. Patching
	2. Channel-Independence
	3. 多尺度时序编码（Two-Level Aggregation）
	4. PEFT（LoRA + LayerNorm Tuning）——参数高效微调
让预训练LLM（GPT-2等）高效对齐时序模态，在全样本长期预测和少样本场景全面超越SOTA，同时成为强表示学习器。

### 1. 动机 & 核心问题

- 时序预测（尤其是长期、多变量）需要大量标注数据，传统从零训练模型（DLinear、PatchTST、FEDformer）在少样本下表现差。
- LLM在NLP/CV上有强大**表示学习 + 少样本/零样本**能力，但直接用于时序面临两大挑战：
    - **模态不匹配**：LLM处理离散token，时序是连续数值 + 多尺度模式（秒/天/季节）。
    - **缺乏时序先验**：LLM预训练在文本上，无法天然捕捉趋势、周期等时序特性。
- 现有方案（如One Fits All、TIME-LLM）要么冻结LLM（效果有限），要么直接微调（易破坏预训练知识）。

**LLM4TS目标**：用**最小参数更新 + 系统对齐**激活LLM在时序上的潜力，实现**数据高效（data-efficient**预测。

### 2. 核心框架（LLM4TS整体结构）

![[LLM4TS.png]]

**输入处理**（继承PatchTST并增强）：

- **Instance Normalization (IN) + RevIN**：缓解分布漂移。
- **Channel-Independence (CI)**：多元序列拆成多个单变量独立处理。
- **Patching**：序列切成固定长度Patch（token化），保留局部语义，降低序列长度。
- **嵌入层**：
    - Token编码：1D Conv（代替词查找表，处理向量输入）。
    - Positional Encoding：可训练查找表。
    - **多尺度时序编码（Two-Level Aggregation，核心创新）**：
        - Level 1：每个时间戳嵌入多种时序属性（秒、分钟、小时、天、周、月、年、节假日等），用查找表嵌入后**求和**。
        - Level 2：Patch级池化（取第一个时间戳），得到Patch级时序嵌入。
        - 最终嵌入 = Token + Positional + Temporal。

**两阶段微调**（核心贡献）：

1. **Time-Series Alignment Stage（时序对齐阶段）**：
    - 目标：让LLM学会“读懂”时序Patch（像聊天模型的SFT）。
    - 任务：**自回归预测下一个Patch**（autoregressive objective）。
    - **PEFT**（参数高效微调）：
        - LoRA：只在注意力Q/K矩阵加低秩适配器。
        - LayerNorm Tuning：只训LayerNorm的仿射参数。
    - 大部分LLM参数冻结（保留预训练知识），只训~1.5%参数。
2. **Forecasting Fine-Tuning Stage（预测微调阶段）**：
    - 先**Linear Probing**（只训输出层）。
    - 再**LP-FT**（Linear Probing + Full Tuning，逐步解冻）。
    - 损失：MSE + RevIN反归一化输出最终预测。

**输出**：LLM输出展平 → 线性投影 → RevIN反归一化 → 预测序列。

**参数效率**：可训参数仅3.4M（GPT-2总参数85M的~4%），远低于从零训练模型。

### 3. 实验结果（7个主流数据集）

- **全样本长期预测**：在Weather、Traffic、Electricity、ETTh1/2、ETTm1/2上MSE/MAE全面SOTA，优于PatchTST、DLinear、FEDformer等。
- **少样本（5%/10%数据）**：排名第一，甚至用5%数据就打败多数全样本基线；远超GPT4TS、TIME-LLM在少样本下的表现。
- **表示学习**（线性探针任务）：优于TS2Vec、TNC等自监督方法6%+，证明LLM对齐后是强表示学习器。
- **效率**：训练/推理更快（参数少），Scaling Law成立（更大LLM效果更好）。

**关键观察**：

- 两阶段对齐 + 多尺度编码贡献最大。
- 冻结预训练权重至关重要（从零训练性能下降17%+）。
- 在少样本下优势最明显（LLM的先验知识被成功激活）。

### 4. 消融与分析

- 去掉Alignment阶段 / 多尺度编码 / PEFT：性能显著下降（少样本更严重）。
- LP-FT策略优于单纯Linear Probing或Full Tuning。
- 多尺度时序嵌入对长预测 horizon 帮助最大。

### 5. 与前文论文对比（Take-Home Message）

- **vs PatchTST / One Fits All**：继承Patching+CI，但加了**多尺度时序编码**和**系统两阶段对齐**，少样本/表示学习更强。
- **vs TIME-LLM**：TIME-LLM是“零微调 + 重编程 + Prompt”（适合零样本，部署最轻）；LLM4TS是“轻微调 + 自回归对齐 + 多尺度”（全样本更优，少样本也优秀）。
- **vs GPT4TS**：LLM4TS更系统（两阶段 + 多尺度 + PEFT），性能全面领先。

**一句话总结**： LLM4TS证明**“先用自回归对齐时序 + 多尺度嵌入 + PEFT微调”是让预训练LLM高效服务时序预测的正确路径**，在全样本和少样本场景都实现了真正的数据高效SOTA，为后续LLM+时序Foundation Model提供了实用框架。

这篇论文和TIME-LLM、One Fits All一起，构成了2023-2024年LLM时序预测的“三驾马车”：重编程（TIME-LLM）、冻结+微调（One Fits All）、两阶段对齐+PEFT（LLM4TS）。
