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
1. 输入HBM中的矩阵：$$Q,K,V\in\mathbb R^{N\times d}$$
2. 计算S写入HBM：$$S=QK^T\in \mathbb R^{N\times N}$$
3. 读取S，计算P，写入HBM：$$P=softmax(S)\in \mathbb R^{N\times N}$$
4. 读取P，V，计算O，写入HBM：$$Output=PV\in \mathbb R^{N\times d}$$
5. 返回Output

---

- FLASHAttention
![[FLASHAttention1.png]]

- 分块
设分块后第i块$X^{(i)}\in \mathbb R^{B\times d}$的的某行：$$x^{(i)}\in \mathbb R^{B}$$
则块内:$$m(x^{(i)})=max_j(x_j^{(i)})$$计算：$$f(x^{(i)})=[e^{x_1-m(x)}...e^{x_B-m(x)}]$$和：$$l(x^{(i)})=\sum_j f(x)_j$$则：$$softmax(x^{(i)})=\frac{f(x)}{l(x)}$$对于多个块，合并后的原向量行(2个为例)：$$x=concat[x^{(1)},x^{(2)}]$$有：$$max(x)=max_i(m(x^{(i)}))$$且（相当于用e指数系数重新规范化f）:$$f(x)=[...e^{m(x^{(i)})-m(x)}*f(x^{(i)})...]$$和（重新规范化l）:$$l(x)=\sum_i e^{m(x^{(i)})-m(x)}*l(x^{(i)})$$
$$softmax(x)=\frac{f(x)}{l(x)}$$


---

- 算法：
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
			- $\hat P_{ij}=e^{S_{ij}-\hat m_{ij}}\in \mathbb R^{B_r\times B_c}$
			- $\hat l_{ij}=rowsum(\hat P_{ij})\in \mathbb R^{B_r}$
			- $m_i^{new}=max(m_i,\hat m_{ij})$,更新最值
			- $l_i^{new}=e^{m_{i}- m_{i}^{new}}l_i+e^{\hat m_{ij}- m_{i}^{new}}\hat l_{ij}$
			- $O_i^{new}=diag(l_i^{new})^{-1}(diag(l_i)e^{m_{i}- m_{i}^{new}}O_i+e^{\hat m_{ij}- m_{i}^{new}}\hat P_{ij}V_j)$
			- 更新$O_i,l_i,m_i$为new版本到HBM
		- end for
	- end for
	- Return O

---

# FLASHAttention-2

- 算法：减少非矩阵运算的FLOPs（现代GPU对矩阵运算优化良好）

- FLASH1如下：
![[FLASH2.png]]

1. 原：$$O^{(2)}=diag(\frac{l^{(1)}}{l^{(2)}})^{-1}O^{(1)}+diag({l^{(2)}})^{-1}e^{S^{(2)}-m^{(2)}}V^{(2)}$$我们改为使用未伸缩的$O^{(2)}$,最后一步才使用l将其变为输出：$$\hat O^{(2)}=diag({l^{(1)}})^{-1}O^{(1)}+e^{S^{(2)}-m^{(2)}}V^{(2)}$$
2. 反向过程，不用分别保存$max{m^{(j)}}$和$e^{l^{(j)}}$,只需:$$logsumexpL^{(j)}=m^{(j)}+log(l^{(j)})$$


---

- forward算法：
	- 输入：$Q，K，V\in \mathbb R^{N\times d}$（HBM），需要大小为M的片上SRAM
	- 块大小:$B_c=\lceil \frac{M}{4d}\rceil$,$B_r=min(\lceil \frac{M}{4d}\rceil,d)$
	- 初始化：$O=(0)_{N\times d},l=(0)_N,m=(-\infty)_N$,在(HBM)
	- O，logsumexp L，Q分成$T_r=\lceil \frac{N}{B_r}\rceil$块;K,V分成$T_c=\lceil \frac{N}{B_c}\rceil$块
	- $for 1\le i \le T_c do$ :
		- 从HBM加载$Q_i$
		- 初始化：$O_i^{(0)}=(0)_{B_r\times d},l_i^{(0)}=(0)_{B_r},m_i^{(0)}=(-\infty)_{B_r}$,在(HBM)
		- $for 1\le j \le T_r do$ :
			- 从HBM加载$K_j,V_j$
			- $S_{i}^{(j)}=Q_iK^T_j\in \mathbb R^{B_r\times B_c}$
			- $m_{ij}=max(m_i^{(j-1)},rowmax(S_{i}^{(j)})\in \mathbb R^{B_r})$
			- $\hat P_{ij}=e^{S_{ij}- m_{ij}}\in \mathbb R^{B_r\times B_c}$
			- $l_{ij}=e^{m_i^{(j-1)}-m_i^{(j)}}l_i^{(j-1)}+rowsum(\hat P_{ij})\in \mathbb R^{B_r}$
			- $l_i^{new}=e^{m_{i}- m_{i}^{new}}l_i+e^{\hat m_{ij}- m_{i}^{new}}\hat l_{ij}$
			- $O_i^{{j}}=diag(e^{m_i^{(j-1)}-m_i^{(j)}})^{-1}O_i^{(j-1)}+\hat P_{i}^{(j)}V_j$
			- 更新$O_i,l_i,m_i$为new版本到HBM
		- end for
	- end for
	- Return O

---
