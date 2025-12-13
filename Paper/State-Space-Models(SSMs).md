# **1.HIPPO**

- 思路：对输入信号进行在线函数逼近
---
### 注：论文的函数逼近

1. 对输入$f(t)$近似函数为$g(t)$
2. 给定测度 $\mu^t$,定义误差$$||f-g||^2_{\mu^t}=<f-g,f-g>_{\mu ^t}=\int{(f(s)-g(s))^2d\mu^t(s)}$$最小化该误差范数的近似函数为最佳近似
3. 其中，给定近似函数$g(t)$在n维正交空间（）中的基为$P_k(t)，k=0：n$，则令$f$该基下的投影为$$C_k=<f_{<t},P_{k}^{(t)}>_{\mu^t}=\int f(s)P_k(s)d\mu^{t}(s)$$
4. 那么定义算子$$proj(f_{\le t})= \sum_{k=0}^{n-1}C_kP_k^t$$
5. 则近似函数:$$g(t)=min||f-g||^2_{\mu^t}=proj(f)$$
---
- 将记忆看成: *输入在多项式基下的投影系数*
- 建立投影系数随时间满足线性 ODE：$$\frac{d}{dt}c(t)=A(t)c(t)+B(t)f(t)$$

- 离散化：给定$\Delta t$,使用ZOH，双线性变换近似：$$\bar A=e^{(A\Delta t)},\bar B=\int_0^{\Delta t}{e^{As}Bds}$$
- 离散化后:$$c_t=\bar A_tc_{(t-1)}+\bar B_tf_t$$
- 论文论证了测度$\mu ^t(x)=legS$下特定结构A矩阵具有长期记忆能力

![[hippo.png|650]]

# **2.LSSL**

- 定义线性状态空间层
- 给定模型： $$\dot x(t)=Ax(t)+Bu(t)$$$$y(t)=Cx(t)+Du(t)$$
- 给定$\Delta t$离散模型：$$x_t=\bar Ax_{t-1}+\bar Bu_t$$$$y_t=Cx_t+Du_t$$
- LSSL层：对于给定HIPPO形式的矩阵A，模型学习输入映射B、输出映射C，（D）。其中隐藏状态$x_t$（可以理解为输入信号在某正交基上的投影系数）为长期记忆。
---
### 卷积理解
- 展开迭代：$$x_{-1}=0$$$$x_0=\bar Ax_{-1}+\bar Bu_0=\bar Bu_0$$$$x_1=\bar Ax_0+\bar Bu_1=\bar A(\bar Bu_0)+\bar Bu_1$$$$...$$$$x_t=\sum_{i=0}^{t}=\bar A^i\bar Bu_{t-i}$$$$y_t=Cx_t+Du_t=\sum _{i=0}^{t}(C\bar A^i\bar B)u_{t-i}+Du_t$$
- 记：$$k_l(A,B,C)=(CB,CAB,CA^2B,...,CA^tB)$$为卷积核，则：$$y_t=k_l(\bar A,\bar B,\bar C)*u_t+Du_t$$
- 即为卷积形式。
- ---
### RNN gate理解：
- 一阶情况：$$\dot x(t)=-x(t)+u(t)$$离散化：$$x_t=x_{t-1}+\Delta t(-x_t+u_t)$$即：$$x_t=\frac{1}{1+\Delta t}x_{t-1}+\frac{\Delta t}{1+\Delta t}u_t$$与Rnn gata对比：$$x_t=(1-g_t)x_{t-1}+g_tu_t$$可以认为:
- 模型对离散化参数$\Delta t$的学习可以看成RNNs的门控机制
- 注:Δt 实现上常见选项：固定超参 / 可学习标量（全局）/ per-dimension 可学 / per-head 可学。RNN 的门可看作离散化步长引发的“更新率”，但 RNN 门通常是输入相关的函数，而单纯学习 Δt 得到的是时间尺度的可调但_不是_输入相关的门。
---
# **3.S4**

- 解决模型计算效率问题:如何快速计算卷积核$$k_l(A,B,C)=(CB,CAB,CA^2B,...,CA^tB)$$
- 提出，模型A矩阵（所有的HiPPO矩阵）可以NPLR分解：$$A=V\Lambda V^*-PQ^T=V(\Lambda -(V^*P)(V^*Q)^*) V^*$$其中V为酉矩阵，P、Q为低秩矩阵（r=1\2）。
- 这样，卷积核K的计算即可快速得到(频域迂回法，直接计算K(w)采用iFFT加速)

## **4.S6(SSM+Selection)**

- 前述模型都为LTI模型，参数A不随时间改变，则缺乏内容感知，不能对输入进行选择
- 思路：让$\Delta t,\bar A,\bar B,\bar C$成为输入的函数。

![[s6-t.png]]

- 核心:$$S_B(x)=linear_N(x)$$$$S_C(x)=linear_N(x)$$$$S_\Delta(x)=Broadcast_D(linear_1(x))$$这里先将x映射到1维，作为当前输入的选择控制，再广播到D维（选择性同时对当前时刻的所有输入做出同样动作）
$$\Delta=softplus(S_\Delta(x))$$其中，论文提到，由于$\bar A=e^{A\Delta}$则让$\Delta$成为x的函数即可让$\bar A$成为x的函数，同理B。

- 模型结构如下:

![[s6-a.png]]

- 结合门控MLP,利用SSM模块捕捉长序依赖

# **5.Mamba-2**

- 在选择性SSM基础上进一步 简化A矩阵:$$A_t=a_tI$$这里，将A简化为标量乘以单位阵形式，且丢弃了低秩项。牺牲了模型表达能力但便于加速计算和引入多头。
- 模型表示为:$$y_t=\sum_{s=0}^{t}C_t^TA_{t:s}^{\times}B_sx_s$$其中$$A_{t:s}^{\times}=A_tA_{t-1}...A_{s+1}$$记：$$y=SSM(A,B,C)(x)=Mx$$其中$$M_{ji}=C^T_jA_j...A_{i+1}B_i$$为一个下三角矩阵。
---
### Semiseparable Matrices

- def:N-Semiseperate Matrices下三角阵，每个子矩阵的最高秩为N
- Sequential semisepqrable(SSS):可以被写成以下形式的矩阵$$M_{ji}=C^T_jA_j...A_{i+1}B_i$$
- 则原SSM模型的优化问题可以转换为SSS矩阵计算问题
- 1-SS Matrices:$$M=1SS(a_{0:T})=\begin{bmatrix}1\\a_1&1\\a_2a_1&a_1&1\\\vdots&\vdots&\vdots&\ddots\\a_{T-1}...a_1&a_{T-1}...a_2&...&a_{T-1}&1\end{bmatrix}$$则论文中的$A_j...A_{i+1}$可以写成$1SS(A)$
---
- 记:$$y_t=\sum_{s=0}^{t}C_t^TA_{t:s}^{\times}B_sx_s$$其中C，B为Q，K，L=1SS(A),x为V：
$$Y_{t,p}=\sum _s\sum _nL_{t,s}Q_{t,n}K_{s,n}V_{s,p}$$
---
### SSD-linear（Recurrent）

1. 计算Z$$Z_{s,p,n}=V_{s,p}K_{s,n}(s固定外积)$$
2. 计算H$$H_{t,p,s}=\sum _sL_{t,s}Z_{s,p,n}=\sum _sL_{t,s}V_{s,p}K_{s,n}$$
3. Y=$$Y_{t,p}=\sum _nQ_{t,n}H_{t,p,n}=\sum _s\sum _nQ_{t,n}L_{t,s}K_{s,n}V_{s,p}$$
- 计算复杂度近似线性，瓶颈为2（矩阵乘向量）。

---
### SSD-Quadratic

1. G$$G_{t,s}=\sum _nQ_{t,n}K_{s,n}$$
2. M:逐元素点乘$$M_{t,s}=L_{t,s}\circ G_{t,s}$$
3. Y$$Y_{t,p}=M_{t,s}V_{s,p}=\sum _s\sum _nL_{t,s}Q_{t,n}K_{s,n}V_{s,p}$$
- 理论复杂度o($T^2$),但短序列时很快（矩阵计算）

---

### 线性注意力机制与SSM：

- 注意力机制：$$Attention(Q,K,V)=softmax(QK^T)V$$
- 省去softmax后mask线性注意力：$$Linear-attention(Q,K,V)=L\circ Q(K^TV)$$（L为下三角矩阵，mask）与上述$$Y=(L\circ QK^T)V$$一致。

---

- 模型结构：

![[mamba-2.png|650]]