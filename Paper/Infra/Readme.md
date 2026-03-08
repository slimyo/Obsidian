如果目标是 **AI Infra / ML Systems / LLM 推理系统工程师**，论文阅读和普通算法方向完全不同。

重点不是模型，而是：

GPU  
推理系统  
训练系统  
算子优化

我给你整理了一套 **AI Infra 必读 15 篇论文路线图**（业内很多 ML Systems 工程师基本都读过）。

我按 **学习阶段排序**，从基础 → 推理系统 → 分布式训练。

---

# 一、Transformer基础（必须）

## 1 Transformer架构

📄 **Attention Is All You Need**

这是所有 LLM 的起点。

核心贡献：

Self-attention  
Multi-head attention  
Transformer architecture

重要公式：

Attention(Q,K,V) = softmax(QK^T / √d) V

AI Infra 为什么必须读：

因为：

attention kernel  
KV cache  
FlashAttention

全部围绕它。

---

# 二、LLM推理优化（最重要）

这是 **AI Infra 最核心的论文**。

---

## 2 FlashAttention

📄 **FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness**

核心思想：

减少GPU memory IO

关键优化：

tile attention  
SRAM reuse

效果：

attention memory ↓  
speed ↑

几乎所有推理系统都使用这个思路。

---

## 3 FlashAttention-2

📄 **FlashAttention-2: Faster Attention with Better Parallelism**

改进：

更好的 GPU 并行调度  
warp-level parallelism

---

## 4 vLLM / PagedAttention

📄 **vLLM: Easy, Fast, and Cheap LLM Serving with PagedAttention**

核心创新：

Paged KV cache

解决问题：

显存碎片  
多请求推理

这是现在 **LLM推理系统最关键的论文之一**。

---

# 三、LLM推理系统

## 5 FlexGen

📄 **FlexGen: High-Throughput Generative Inference of Large Language Models with a Single GPU**

核心思想：

CPU + GPU + SSD memory offloading

解决：

显存不足

---

## 6 TurboTransformers

📄 **TurboTransformers: An Efficient GPU Serving System For Transformer Models**

核心：

inference kernel optimization

---

## 7 DeepSpeed Inference

📄 **DeepSpeed Inference: Enabling Efficient Inference of Transformer Models at Unprecedented Scale**

优化：

kernel fusion  
parallel inference

---

# 四、分布式训练系统

训练系统也是 AI Infra 的核心方向。

---

## 8 Megatron-LM

📄 **Efficient Large-Scale Language Model Training on GPU Clusters Using Megatron-LM**

提出：

tensor parallel  
pipeline parallel  
data parallel

三维并行。

可以在 **数千GPU上训练LLM**。

---

## 9 Megatron-Turing NLG

📄 **Using DeepSpeed and Megatron to Train Megatron‑Turing NLG 530B**

训练：

530B参数模型

核心技术：

3D parallelism  
DeepSpeed

展示了 **工业级 LLM 训练系统**。

---

## 10 MegaScale

📄 **MegaScale: Scaling Large Language Model Training to More Than 10,000 GPUs**

核心：

10000 GPU 训练系统

重点是：

系统稳定性  
故障恢复  
straggler mitigation

这是 **超大规模训练系统设计**。

---

# 五、ML系统经典论文

理解 **深度学习框架设计**。

---

## 11 MXNet

📄 **MXNet: A Flexible and Efficient Machine Learning Library for Heterogeneous Distributed Systems**

介绍：

autograd  
tensor computation  
distributed training

是很多框架设计思想的来源。

---

## 12 TensorFlow (经典系统论文)

📄 **TensorFlow: Large-Scale Machine Learning on Heterogeneous Systems**

关键思想：

computation graph  
distributed execution

---

## 13 PyTorch (系统设计)

📄 **PyTorch: An Imperative Style, High-Performance Deep Learning Library**

核心：

dynamic graph  
autograd  
tensor library

---

# 六、GPU kernel优化

## 14 Online Softmax

📄 **Online Normalizer Calculation for Softmax**

FlashAttention 的基础。

---

## 15 FlashAttention-3

📄 **FlashAttention-3: Fast and Accurate Attention with Asynchrony and Low Precision**

最新 attention kernel。

---

# 七、最重要的 5 篇（强烈建议先读）

如果时间有限，先读这五篇：

Attention Is All You Need  
FlashAttention  
vLLM (PagedAttention)  
Megatron-LM  
DeepSpeed Inference

原因：

Transformer结构  
attention kernel  
LLM推理  
LLM训练

基本覆盖 **AI Infra 核心技术栈**。

---

# 八、最推荐阅读顺序

建议按这个顺序：

1 Attention Is All You Need  
2 FlashAttention  
3 FlashAttention-2  
4 vLLM  
5 FlexGen  
6 DeepSpeed Inference  
7 Megatron-LM  
8 MegaScale

读完这几篇，你会基本理解：

LLM是怎么训练的  
LLM是怎么推理的  
GPU是怎么优化的