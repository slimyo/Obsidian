**《Transformers are RNNs: Fast Autoregressive Transformers with Linear Attention》论文解读** （ICML 2020，arXiv:2006.16236，作者：Angelos Katharopoulos, Apoorv Vyas, Nikolaos Pappas, François Fleuret）

**一句话核心思想**： **把标准Transformer的自注意力改成“线性注意力”（Linear Attention），就能把整个Transformer“变成”一个RNN，实现O(N)线性复杂度 + 恒定内存的自回归推理，速度比标准softmax注意力快几千倍，同时性能几乎不掉！**

这篇论文是2020年Transformer高效化领域的里程碑之一，直接解决了“自回归生成时Transformer太慢”的痛点（尤其是长序列），并从理论上证明了**Transformer和RNN本质上可以等价**。

### 1. 背景 & 核心问题

标准Transformer的自注意力复杂度是 **O(N²)**（N=序列长度），这在训练时还能忍受（并行），但在**自回归推理（autoregressive decoding）** 时灾难性：

- 每生成一个新token，都要对当前整个前缀（长度越来越长）重新算一次注意力。
- 内存和时间随序列长度二次增长，生成几千token时直接卡死。

论文目标：**保持Transformer的表达能力，但让自回归推理变成O(1) per token（线性总时间、恒定内存）**。

### 2. 核心创新：Linear Attention（线性注意力）

作者把softmax注意力改写成**核函数（kernel）特征映射的线性点积**，然后利用**矩阵乘法的结合律（associativity）** 把复杂度从O(N²)降到O(N)。

#### 标准注意力回顾

$$V'_i = \sum_{j=1}^N \frac{\mathrm{sim}(Q_i, K_j)}{\sum_{j=1}^N \mathrm{sim}(Q_i, K_j)} ​$$

（softmax时$\mathrm{sim}(q,k) = \exp(q^\top k / \sqrt{d})$）
#### 线性注意力公式（关键改写）

引入正定核函数 $$k(\mathbf{x},\mathbf{y}) = \phi(\mathbf{x})^\top \phi(\mathbf{y}) $$ 和特征映射$\phi$，注意力变成：

$$V'_i = \phi(Q_i)^\top \left( \sum_{j=1}^N \phi(K_j) V_j^\top \right) \bigg/ \phi(Q_i)^\top \left( \sum_{j=1}^N \phi(K_j) \right)$$

向量形式（矩阵乘法结合律）：

$$\bar{V} = \phi(Q) \bigl( \phi(K)^\top V \bigr), \quad \bar{Z} = \phi(Q) \bigl( \phi(K)^\top \bigr)$$
$$V' = \bar{V} \oslash \bar{Z}$$

**关键技巧**：两个大和 $\sum \phi(K_j) V_j^\top$​ 和 $\sum \phi(K_j)$ 可以**预先算好**并复用，所有query共享同一个“汇总状态”，复杂度瞬间变成O(N)。

**论文推荐的特征映射**（最常用）：

$$\phi(x) = \mathrm{ELU}(x) + 1$$

（保证非负、梯度稳定；比ReLU好，避免零梯度问题）

### 3. 最大洞见：Transformer ≡ RNN（自回归下的迭代形式）

对于**因果掩码（causal masking）** 的自回归场景，公式进一步简化为**累积状态更新**：

定义状态：

$$S_i = \sum_{j=1}^i \phi(K_j) V_j^\top, \quad Z_i = \sum_{j=1}^i \phi(K_j)$$

则第i步输出：

$$V'_i = \phi(Q_i)^\top S_i \big/ \phi(Q_i)^\top Z_i​$$

**迭代更新（RNN形式）**：

$$S_i = S_{i-1} + \phi(K_i) V_i^\top, \quad Z_i = Z_{i-1} + \phi(K_i)$$
$$y_i = f_l \bigl( \phi(x_i W^Q)^\top S_i \, (\phi(x_i W^Q)^\top Z_i)^{-1} + x_i \bigr)$$
- $S_i$​ 和$Z_i$ 就是RNN的“隐藏状态”（hidden state）。
- 每步只需O(1)更新（常数时间、常数内存），**完全不需要重新算整个序列**。
- 这就是论文标题“Transformers are RNNs”的由来。

### 4. 复杂度对比

- **标准Transformer**：训练O(N²)，自回归推理O(N²) per token → 总O(N³)
- **Linear Transformer**：
    - 训练：O(N)（保持并行）
    - 自回归推理：**O(1) per token**，总O(N)
- 实测速度提升：**MNIST上317×，CIFAR-10上4462×**，最长序列可达**4000×**加速。

### 5. 实验结果（简洁版）

- **任务**：图像生成（MNIST/CIFAR-10，按像素序列自回归）、语音识别（WSJ）。
- **性能**：Linear Transformer 与标准softmax Transformer**几乎持平**（甚至在等训练时间下略胜），但速度碾压。
    - MNIST bits/dim：0.644（Linear） vs 0.621（softmax）
    - CIFAR-10：3.40 vs 3.47
    - WSJ PER：8.08（优于Reformer的9.33）
- **对比**：Reformer（LSH注意力）、Stateful softmax（缓存版，但仍二次复杂度）。
- **消融**：ELU特征映射最稳；多项式核（degree=2）也有效但略贵。

图示（论文Figure 1）：Linear版本时间/内存随序列长度**线性增长**，softmax**二次爆炸**。

### 6. 论文的Take-Home Knowledge（最大启发）

1. **注意力不一定需要softmax**：用核特征映射 + 结合律就能线性化，性能不丢。
2. **Transformer本质上是RNN**：自回归时完全可以迭代实现，这为后续Linear Attention、RetNet、RWKV等“RNN-like Transformer”奠定了理论基础。
3. **自回归推理才是真正的瓶颈**：训练时二次复杂度可忍，但生成时必须解决。
4. **常数内存 + 线性时间**：让Transformer真正能处理超长序列（几万token）成为可能。

**一句话总结论文意义**： 这篇工作第一次从数学上证明了“**Transformer可以用RNN的方式高效实现自回归**”，直接开启了“高效Transformer”时代（后续的Performer、Linear Transformer、FlashAttention等都受其影响）。