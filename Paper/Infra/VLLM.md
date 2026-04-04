# vLLM
- 现有LLM服务系统管理KVcache不高效
	- 原因：KVcache在内存空间中临近存放
	![[vLLM1.png|1000|800]]
	- 导致不高效：内存分片、内存共享问题
		- reserved slots:给未来的词元
		- internal fragmentation:为存放KVcache，预留空间一过长（输入的最大长度）。且没用到的内存块不能充分利用。
		- external fragmentation:由于不同序列的KVcahce存放不相邻，序列间可能可以共享的KV内存不能共享使用
- 解决：PagedAttention
	- 将请求的KVcache分成大小相同的多个块
	- 类比：块--页，tokens-bytes，request-进程
- LLM服务：
	- request：根据提示词prompt给出输出词元token
	- sequence：prompt和token拼接结果
	- 服务响应请求的两个阶段：
		- prompt：根据提示词计算首个token
		- autoregressive generation：根据目前sequence自回归计算结果
	- batching techniques：批处理
		- 核心：合并多个请求共享模型权重，分摊权重移动的开销。
	- 内存挑战：
		- KV cahce很大
		- decoding算法复杂
		- 未知输入输出长度的调度：KVcache可能耗尽缓冲，系统要决策删除或交换部分缓冲


- PagedAttention
	- 允许将临近的KV以非临近方式存储到内存
	- 将sequence的KVcache分割成KVblocks，每个包含固定数量token的KV向量
	- 计算：假设block大小为B
$$\begin{align*}
K-block:K_j&=[k_{(j-1)B+1}...k_{jB+1}]\\
V-block:V_j&=[v_{(j-1)B+1}...v_{jB+1}]\\
A_{ij}&=\frac{exp(q_i^TK_j/\sqrt{d})}{\sum_{t=1}^{\lceil i/B\rceil}exp(q_i^TK_t1/\sqrt{d})}\\
o_{ij}&=\sum_{j=1}^{\lceil i/B\rceil}V_jA_{ij}^T
\end{align*}$$

- KVcache管理
	- 如图，使用Block Table管理逻辑块和物理块的映射
![[KVcacheManerger.png]]

- 其他解码器情况
	- 并行采样：对于一个提示词，模型产生多个输出
		- 1个request产生多个sequences
		- 多数KVcache采用共享、最后一个逻辑块采用copy-on-write机制管理
			- 内部block维护参考计数
		- ![[ParallelSampling.png]]
	- BeamSearch集束搜索
		- 在每一步会保留 `k`个最可能的候选序列
		- copy-on-write
		- k=4的情况：
			- 虚线前，各个候选都共享Block0（reference计数=4），然后分成两份 
			- 虚线后，top-k都来自候选1，2所在分支，故将多余block（reference计数=0）释放
		- ![[BeamSearch.png]]
	- shared-prefix共享前缀
		- 系统提示词（System Prompt）经常被重复使用
		- 很多提示词有共享前缀
		- 可以提前缓存共享前缀的KVcache减少计算
		- ![[sharedprefix.png]]

- Scheduling and Preemption调度与优先权
	- 请求拥堵，超过系统容量：
		- 调度原则：FCFS（first-come-first-serve） 与公平性
	- LLM 服务与传统 Web 服务不同，输入长度差异大，且**输出长度无法预知**。随着生成进行，KV Cache 会不断膨胀，直至耗尽 GPU 物理块。这引出了两个核心问题：
		1. **该驱逐谁？**（Eviction Policy）
			1. 驱逐策略：All-or-Nothing对于一个sequence的block要么全驱逐走，要么不驱逐
			2. 一个request生成的多个sequence：以**序列组（Sequence Group）**为单位（例如 Beam Search 中的一组候选序列）。组内序列因共享内存，必须**整体调度、整体抢占**（Gang-scheduled）。
		2. **驱逐后如何恢复？**（Recovery Mechanism）
			1. 对比两种机制：

|机制|原理|适用场景|
|---|---|---|
|**Swapping（交换）**​|将 GPU 物理块拷贝至 **CPU 内存**（充当 Swap 空间）。需时再拷回。|CPU-GPU 带宽较高的场景；长上下文且重复利用率高。|
|**Recomputation（重算）**​|不存 KV Cache，需要时把“提示+已生成Token”作为新 Prompt **重跑一次**。|计算力强于 IO 带宽的场景；短序列。|
- **关键细节**：
- **Swap 空间有界**：CPU 侧的 Swap 空间大小**不超过 GPU 的 KV Cache 总容量**，防止主机内存被拖垮。
- **重算加速**：重算时可将历史 tokens 与 Prompt 拼接，在**一次前向传播**中全部算完，而非逐 token 重推，效率更高。

- 分布式执行（Distributed Execution）
	1. 背景：大模型必须分片
		- 175B 等大模型单卡装不下，必须采用 **Model Parallelism**（如 Megatron-LM 风格）切分到多卡。
	2. 架构：集中式管理 + SPMD
		- vLLM 采用**集中式调度器（Scheduler**统一管理 KV Cache，而非每张卡各自为政：
			- **逻辑映射共享**：所有 GPU Worker 共享**同一套**“逻辑块 → 物理块”映射表。
			- **物理存储分片**：虽然映射表一样，但每张 GPU 只存储**自己分片**的那部分 Attention Heads 的 KV 数据。
	3. 执行流程（SPMD（single-program-mutiple-data） 模式）
		1. **Scheduler 广播**：调度器准备好 Input Token IDs 和 Block Table，广播给所有 Worker。
		2. **Worker 并行计算**：各 Worker 根据 Block Table 读取本地的 KV Cache 分片，执行 Attention 计算。
		3. **隐式同步**：在 Attention 层通过 **All-Reduce**​ 同步中间结果（由 Megatron-LM 框架层负责，vLLM 不介入）。
		4. **回传采样**：Worker 将采样出的新 Token 回传给 Scheduler。
	- 优势：解耦内存与计算
		- **Worker 无状态**：GPU Worker 不参与内存管理决策，只需在每步解码开始时接收 Scheduler 的指令。
		- **无额外同步**：Worker 之间**不需要**为内存分配进行额外同步，极大降低了分布式复杂度。