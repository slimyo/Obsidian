《One Fits All: Power General Time Series Analysis by Pretrained LM》 （arXiv:2302.11939v6，NeurIPS 2023，阿里巴巴团队）

- 背景：大模型在NLP、CV领域取得重要成就
	- 在时序处理领域，由于数据集小，无法从头训练大模型
- **解决方案**：直接借用GPT-2（或BERT、BEiT）这些已经用亿级token训好的模型，只改很少的参数，就让它“跨模态”做时间序列。
- 模型：
- **冻结部分**（完全不动）：预训练Transformer的所有Self-Attention块 + FeedForward层（这就是GPT-2/BERT的核心知识）。
- **只训练的部分**（极少参数）：
    - 输入Embedding层（把时间序列数值投影到模型的hidden dim）。
    - 位置编码 + LayerNorm。
    - 输出线性层。
- **额外技巧**：
    - **Instance Normalization + Reverse Instance Norm**（Kim et al. 2022）：先标准化，再加回均值方差，减少分布漂移。
    - **Patching**（PatchTST）：把连续时间步拼成一个patch，当作一个token，极大增加输入长度，同时减少冗余。
    - GPT2(6)表示只用GPT-2的前6层。

- 结果：**可训练参数只占总参数的4.6%~6.12%**，训练和推理速度反而比很多小模型更快（HuggingFace实现优化得好）。


![[FPT.png]]

- 实验：作者在7大主流任务上全面对比SOTA（TimesNet、PatchTST、DLinear、FEDformer等），用**同一个GPT2-backbone FPT**：

|任务|FPT表现|对比SOTA（TimesNet）|备注|
|---|---|---|---|
|**填补（Imputation）**|平均MSE下降4.1%（ETTn1下降11.5%）|显著优于|6个数据集|
|**分类**|74.00%准确率|优于73.60%|10个UEA数据集|
|**异常检测**|F1=86.72%（+1.7%）|优于|5个工业数据集|
|**长期预测**|平均MSE下降9.3%|优于|8个数据集|
|**短期预测（M4）**|与TimesNet/N-BEATS相当|优于多数Transformer|-|
|**少样本预测（10%数据）**|平均MSE下降33.3%（vs TimesNet）|大幅领先|-|
|**零样本预测**|平均指标优于DLinear、TimesNet、PatchTST|部分数据集接近N-BEATS|跨数据集零样本|

- 理解模型：理论解释：为什么预训练LM能直接用于时间序列？（Section 8）
- 作者发现：
- 启发：预训练Transformer的**token相似度**随层数加深而急剧升高（最后一层>0.9）。
- 测试：**随机初始化**的模型上述相似度一直很低（0.1~0.2）。
- 把Self-Attention替换成**PCA**后，token相似度模式几乎一模一样。
	- 数学推理：------
	- **理论证明**（Lemma 8.1 + Theorem 1）： 自注意力层的梯度结构，使得在训练时最小化梯度范数，等价于让输入模式在**主成分方向上投影**（即PCA）。 数学上，Self-Attention学到的权重矩阵A，最优解正好是输入协方差矩阵XᵀX的前m个最大特征向量（m是注意力头秩）。
	- **通俗理解**：预训练LM的Self-Attention其实学会了一种**通用的“主成分提取”操作**，它不依赖文本，而是对任何结构化输入（包括时间序列）都有效。这解释了为什么“一个冻结的Transformer就能通用”——它本质上是个**通用计算引擎**（universal computation engine）。


- **总结**：
- 时间序列领域不再需要从零训大模型，也不需要海量时序数据。
- 以后做时序任务，可能只需要加载HuggingFace的GPT-2，改个Patching + Instance Normalization + 输入Embedding，就能直接用。

- **局限**（论文承认）：
- 零样本在个别数据集仍略逊N-BEATS（缺少meta-learning）。
- 理论分析还在早期，后续可从n-gram视角继续挖掘。