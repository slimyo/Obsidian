## FLASHAttention

- 特点：存储器访问少
- 目标：减少attention矩阵在HBM（GPU Hiagh Bandwidth Memory）的r/w.
- 要求：
	- softmax reduction不access to the 整个输入
	- 反向过程不存储大的中间attention矩阵
- 做法：
	- 重构attention计算，拆成块
	- 存储前向softmax normalization factor，片上快速计算attention矩阵，防止HBM的读写
- Kernel fusion:多个操作要用同一个输入——只从HBM读取一次，然后继续多个处理；
	- 模型训练时，中间值仍然要写入HBM——反向过程需要。这减少了原生Kernel fusion的效率

---
- Standard Attention Implementation

算法：对于长度为N的序列：

- farward pass
1. 输入HBM中的矩阵：$$Q,K,V\in\mathbb R^{N\times d}$$
2. 计算S写入HBM：$$S=QK^T\in \mathbb R^{N\times N}$$
3. 读取S，计算P，写入HBM：$$P=softmax(S)\in \mathbb R^{N\times N}$$
4. 读取P，V，计算O，写入HBM：$$Output=PV\in \mathbb R^{N\times d}$$
5. 返回Output

---
- Backward pass
$$\begin{align*}
\displaystyle 
dV&=P^TdO\in \mathbb R^{(N\times d)}\\
dP&=dOV^T\in \mathbb R^{(N\times N)}\\
dS&=dsoftmax(dP)\in \mathbb R^{(N\times N)}\\
dQ&=dSK\in \mathbb R^{(N\times d)}\\
dK&=QdS^T\in \mathbb R^{(N\times d)}\\
\end{align*}$$进一步推导，行向量p,s有：$$\begin{align*}
p&=softmax(s)\\
J&=[J_{ij}]\\
J_{ij}&=\frac{\partial p_i}{\partial s_j}=\begin{cases}\frac{e^{s_{i}}}{\sum_k e^{s_k}}-(\frac{e^{s_i}}{\sum_k e^{s_k}})^2,i=j\\
-\frac{e^{s_i}*e^{s_j}}{(\sum_k e^{s_k})^2},i\neq j\end{cases}\\
J&=diag(p)-pp^T\\
\end{align*}$$于是:$$\begin{align*}
ds&=(diag(p)-pp^T)dp\in \mathbb R^{(1\times N)}\\
diag(p)dp&=p\odot dp\\
pp^Tdp&=p*rowsum(p\odot dp)
\end{align*}$$则：$$ds=p\odot dp-p*rowsum(p\odot dp)$$由于：$$p\circ dp\leftarrow \rightarrow O\circ dO$$

---

- FLASHAttention
![[FLASHAttention1.png]]

- 分块
设分块后第i块$X^{(i)}\in \mathbb R^{B\times d}$的的某行：$$x^{(i)}\in \mathbb R^{B}$$
则块内:$$m(x^{(i)})=max_j(x_j^{(i)})$$计算中间结果：$$f(x^{(i)})=[e^{x_1-m(x)}...e^{x_B-m(x)}]$$和**缩放因子**：$$l(x^{(i)})=\sum_j f(x)_j$$则：$$softmax(x^{(i)})=\frac{f(x)}{l(x)}$$对于多个块，合并后的原向量行(2个为例)：$$x=concat[x^{(1)},x^{(2)}]$$有：$$max(x)=max_i(m(x^{(i)}))$$且（相当于用e指数系数重新规范化f）:$$f(x)=[...e^{m(x^{(i)})-m(x)}*f(x^{(i)})...]$$和（重新规范化l）:$$l(x)=\sum_i e^{m(x^{(i)})-m(x)}*l(x^{(i)})$$
$$softmax(x)=\frac{f(x)}{l(x)}$$

---

- farward算法：
	- 输入：Q，K，V（HBM），需要大小为M的片上SRAM
	- 块大小:$B_c=\lceil \frac{M}{4d}\rceil$,$B_r=min(\lceil \frac{M}{4d}\rceil,d)$
	- 初始化：$O=(0)_{N\times d},l=(0)_N,m=(-\infty)_N$,在(HBM)
	- O，l，m，Q分成$T_r=\lceil \frac{N}{B_r}\rceil$块;K,V分成$T_c=\lceil \frac{N}{B_c}\rceil$块
	- $for 1\le j \le T_c do$ :
		- 从HBM加载$K_j,V_j$
		- $for 1\le i \le T_r do$ :
			- 从HBM加载$O_i,l_i,m_i,Q_i$
			- $S_{ij}=Q_iK^T_j\in \mathbb R^{B_r\times B_c}$
			- $\hat m_{ij}=rowmax(S_{ij})\in \mathbb R^{B_r}$
			- $\hat P_{ij}=e^{S_{ij}-\hat m_{ij}}\in \mathbb R^{B_r\times B_c}$同上述f
			- $\hat l_{ij}=rowsum(\hat P_{ij})\in \mathbb R^{B_r}$
			- $m_i^{new}=max(m_i,\hat m_{ij})$,更新最值
			- $l_i^{new}=e^{m_{i}- m_{i}^{new}}l_i+e^{\hat m_{ij}- m_{i}^{new}}\hat l_{ij}$
			- $O_i^{new}=diag(l_i^{new})^{-1}(diag(l_i)e^{m_{i}- m_{i}^{new}}O_i+e^{\hat m_{ij}- m_{i}^{new}}\hat P_{ij}V_j)$
			- 更新$O_i,l_i,m_i$为new版本到HBM
		- end for
	- end for
	- Return O

---
- backward:
	$$\begin{align*}
	S&=QK^T\in \mathbb R^{(N\times N)}\\
	P&=softmax(S)=\frac{e^{s-m}}{l}\in \mathbb R^{(N\times N)}
	\end{align*}$$再根据标准公式计算。

---
# FLASHAttention-2

- 算法：减少非矩阵运算的FLOPs（现代GPU对矩阵运算优化良好）

- Loop order切换：FA1是外循环K/V块（j）、内循环Q块（i）；FA2改为外循环Q块（i）、内循环K/V块（j）。这样外循环（over Q rows）可以完全并行（不同Q块独立，无需通信），适合GPU thread block并行，提高occupancy。
	- 注意到：$attention_i=softmax(Q_i*K^T)V$即，注意力计算的序列Q不同行结果互不相干，且与K，V每一行有关。
		- 不同Q块可独立计算，不需要跨block通信。
	- GPU有多个Streaming Multiprocessor (SM)，每个SM可以运行一个或多个 thread block（线程块）。
		- **FA1**：主要并行在 batch × heads 维度。一个 thread block 通常负责整个 head，在 block 内部执行“外循环 KV blocks、内循环 Q blocks”。thread block 数量较少，SM occupancy 较低。
		- **FA2**：外循环改为 over Q blocks，每个 thread block **只负责一个固定的 $Q_i$ block**（$Q_i$保持 stationary），内循环遍历所有 KV blocks。不同 Q blocks 的 thread block **完全独立、无需任何 block 间通信**，显著提升 occupancy。
	- warp层面：一个 thread block 通常有 4~8 warps（每个 warp = 32 threads）
		- FA1：把 K 和 V 沿着列方向切分给不同 warp
			- 每个先计算$Q × K_slice^T$（得到部分 S slice），然后写入共享内存
			- warp线程同步
			- 读取部分S,计算部分O,线程同步
		- FA2：把 Q 沿着行方向切分给不同 warp
			- 每warp内迭代计算KV即可
			- 减少warp级同步、共享内存读写

- FLASH1如下：
![[FLASH2.png]]

1. 原：$$O^{(2)}=diag(\frac{l^{(1)}}{l^{(2)}})^{-1}O^{(1)}+diag({l^{(2)}})^{-1}e^{S^{(2)}-m^{(2)}}V^{(2)}$$我们改为使用未伸缩的$O^{(2)}$:$$\hat O^{(2)}=diag({l^{(1)}})^{-1}O^{(1)}+e^{S^{(2)}-m^{(2)}}V^{(2)}$$最后一步才使用l将其变为输出：$$O=diag(l^{(last)})^{-1}\hat O^{(last)}$$
2. 反向过程，不用分别保存${m^{(j)}}$和${l^{(j)}}$,只需:$$logsumexpL^{(j)}=m^{(j)}+log(l^{(j)})$$有：$$P=softmax(S)=\frac{e^{s-m}}{l}=e^{S-L}$$

---

- forward算法：
	- 输入：$Q，K，V\in \mathbb R^{N\times d}$（HBM），需要大小为M的片上SRAM
	- 块大小:$B_c=\lceil \frac{M}{4d}\rceil$,$B_r=min(\lceil \frac{M}{4d}\rceil,d)$
	- 初始化：$O=(0)_{N\times d},l=(0)_N,m=(-\infty)_N$,在(HBM)
	- O，logsumexp L，Q分成$T_r=\lceil \frac{N}{B_r}\rceil$块;K,V分成$T_c=\lceil \frac{N}{B_c}\rceil$块
	- $for( 1\le i \le T_r )do$ :
		- 从HBM加载$Q_i$
		- 初始化：$O_i^{(0)}=(0)_{B_r\times d},l_i^{(0)}=(0)_{B_r},m_i^{(0)}=(-\infty)_{B_r}$,在(HBM)
		- for$(1\le j \le T_c)$do :
			- 从HBM加载$K_j,V_j$
			- $S_{i}^{(j)}=Q_iK^T_j\in \mathbb R^{B_r\times B_c}$
			- $m_{ij}=max(m_i^{(j-1)},rowmax(S_{i}^{(j)})\in \mathbb R^{B_r})$
			- $\hat P_{ij}=e^{S_{ij}- m_{ij}}\in \mathbb R^{B_r\times B_c}$
			- $l_{ij}=e^{m_i^{(j-1)}-m_i^{(j)}}l_i^{(j-1)}+rowsum(\hat P_{ij})\in \mathbb R^{B_r}$
			- $O_i^{{j}}=diag(e^{m_i^{(j-1)}-m_i^{(j)}})^{-1}O_i^{(j-1)}+\hat P_{i}^{(j)}V_j$
		- end for
		- $O_i=diag(l_i^{(T_c)})^{-1}O_i^{T_c}$（最后一个伸缩回去）
		- $L_i=m_i^{(T_c)}+log(l_i^{(T_c)})$
		- 写入$O_i,L_i$到HBM，作为第i个block的O,L
	- end for
	- Return O,logsumexp L

---

- backward算法：
- 初始化:$D=rowsum(dO\circ O)\in \mathbb R^{(d)}$分成$T_r$块，每块$B_r$
- for j:$T_c$
	- for i:$T_r$
		循环内：
	$$\begin{align*}
	S_i^{(j)}&=Q_iK_j^T\in \mathbb R^{(B_r\times B_c)}\\
	P_i^{(j)}&=e^{S_i^{(j)}-L_i}\in \mathbb R^{(B_r\times B_c)}\\
	dV_j&\leftarrow dV_j+(P_i^{(j)})^{T}dO_i\in \mathbb R^{(B_c\times d)}\\
	dP_i^{(j)}&=dO_iV_j^T\in \mathbb R^{(B_r\times B_c)}\\
	dS_i^{(j)}&=P_i^{(j)}\circ (dP_i^{(j)}-D_i)\in \mathbb R^{(B_r\times B_c)}\\
	dQ_i&\leftarrow dQ_i+dS_i^jK_j\in \mathbb R^{(B_r\times d)}\\
	dK_j&\leftarrow dK_j+dS_i^jQ_i\in \mathbb R^{(B_c\times d)}
	\end{align*}$$
---

