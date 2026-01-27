# **Deep Unsupervised Learning Using Nonequilibrium Thermodynamics**

---
## 一、论文解决的**根本问题**

在 2015 年，**无监督生成模型**面临一个核心困难：
> **真实数据分布很复杂，但我们不知道如何构造一个“可训练、可采样”的概率模型。**
当时的主流方法要么：
- **能算似然但不好训练**（RBM、能量模型）
- **好训练但没概率语义**（自编码器）
- **采样困难或不稳定**（早期 GAN）
### 这篇论文的出发点是一个非常“物理”的想法：

> **如果我能把复杂分布，慢慢“热化”成一个简单分布；  
> 那我是否可以学习“反向冷却”的过程，把噪声变回数据？**

---

## 二、论文提出的核心思想

> **构造一个人为设计的、逐步加噪的非平衡过程（forward diffusion），  
> 并用深度网络去学习它的逆过程，从而实现无监督生成建模。**

这是第一次**系统性地提出「扩散式生成模型」**。

---
## 三、它做了三件“以前没人系统做过的事”

### ① 把“生成建模”变成**路径问题**
传统生成模型关注的是：
$$p(x)$$
这篇论文关注的是：
$$x_0→x_1→⋯→x_T​$$
- 数据不是一次性生成的
- 而是**通过一条马尔可夫轨迹慢慢演化出来**
**建模对象从“点分布”变成了“动力学过程”**

---
### ② 引入一个“人为可控、不可逆”的前向过程

论文设计了一个**完全已知、无需学习的 forward process**
- 每一步加一点高斯噪声
- 保证：
    - 数学上可分析
    - 最终收敛到标准高斯    
这一步的关键意义是：
> **把“复杂数据分布”转换成“简单分布 + 过程结构”**

模型不再直接面对复杂分布。

---
### ③ 把学习问题转化为“学逆物理过程”

真正要学的不是数据分布本身，而是：

$$p_\theta(x_{t-1} \mid x_t)$$
也就是：
> **“如果现在是这个状态，往前一步应该长什么样？”**

这一步的本质是：
- 去噪
- 还原信息
- 减少熵
    
 **生成 = 逐步降低熵的过程**

## **四、核心公式推导**

---
- 熵：$$H(X)=-\int p(x)logp(x)dx$$
- 条件熵:$$H(Y|X)=-\int dxdyp(x,y)logp(y|x)$$有$$H(Y|X)=H(X,Y)-H(X)$$
- KL散度（相对熵）:$$D_{KL}(P||Q)=\int dxp(x)log\frac{p(x)}{q(x)}$$
---
- 前向轨迹
- 定义稳定概率分布：$$\pi(y)=\int dy'T_\pi (y|y';\beta)\pi (y')$$转移函数:$$q(x^{(t)}|x^{(t-1)})=T_\pi (x^{(t)}|x^{(t-1)};\beta)$$则前向轨迹为:
$$q(x^{(0...T)})=q(x^{(0)})\prod_{t=1}^{T} q(x^{(t)}|x^{(t-1)})$$
---
- 反向轨迹$$p(x^{(T)})=\pi(x^{(T)})$$且:$$p(x^{(0...T)})=p(x^{(T)})\prod_{t=1}^{T} p(x^{(t-1)}|x^{(t)})$$
---
- 模型概率(从起点$x^{(T)}$高斯噪声开始的，所有生成目标数据$x^{(0)}$的路径积分和)$$p(x^{(0)})=\int dx^{(1...T)}p(x^{(0...T)})$$无法直接求，改为:$$
\begin{align*}
\displaystyle
p(x^{(0)})&=\int dx^{(1:T)}
p(x^{(0:T)})
\frac{q(x^{(1:T)}|x^{(0)})}
{q(x^{(1:T)}|x^{(0)})}\\
&=\int dx^{(1:T)}
q(x^{(1:T)}|x^{(0)})
\frac{p(x^{(0:T)})}
{q(x^{(1:T)}|x^{(0)})}\\
&=\int dx^{(1:T)}
q(x^{(1:T)}|x^{(0)})
\cdotp(x^{(T)})
\prod_{t=1}^T
\frac{p(x^{(t-1)}|x^{(t)})}
{q(x^{(t)}|x^{(t-1)})}
\end{align*}$$我用一个我能采样的分布$q(x^{(1:T)}|x^{(0)})$ 来重写积分

---
- 目标函数(极大似然估计)：$$L=\int dx^{(0)}q(x^{0})log [p(x^{0})]= \mathbb E_{q(x^{(0)})}[\log p(x^{(0)})]$$代入前文等式得:$$=\int dx^{(0)}q(x^{0})\cdot log[\int dx^{(1...T)}q(x^{(1...T)}|x^{(0)})\cdot p(x^{(T)})\prod_{t=1}^{T}\frac{p(x^{(t-1)}|x^{(t)})}{q(x^{(t)}|x^{(t-1)})}]$$应用Jensen不等式：$$L\ge \int dx^{(0...T)}q(x^{(0...T)})\cdot log[ p(x^{(T)})\prod_{t=1}^{T}\frac{p(x^{(t-1)}|x^{(t)})}{q(x^{(t)}|x^{(t-1)})}]=K$$打开log，边缘化无关变量$x^{0:T-1}$：$$K= \int dx^{(0...T)}q(x^{(0...T)})\cdot \sum _{t=1}^{T}log[ \frac{p(x^{(t-1)}|x^{(t)})}{q(x^{(t)}|x^{(t-1)})}]+\int dx^{(T)}q(x^{(T)})log(p(x^{(T)}))$$根据熵定义$H(x)=-\int dxp(x)log(p(x))$，且$p(x^{(T)})=\pi(x^{(T)})$则有:$$K= \sum _{t=1}^{T}\int dx^{(0...T)}q(x^{(0...T)})\cdot log[ \frac{p(x^{(t-1)}|x^{(t)})}{q(x^{(t)}|x^{(t-1)})}]-H_p(x^{(T)})$$拿出第一项,有:$$K= \sum _{t=2}^{T}\int dx^{(0...T)}q(x^{(0...T)})log[ \frac{p(x^{(t-1)}|x^{(t)})}{q(x^{(t)}|x^{(t-1)})}]+\int dx^{(0...T)}q(x^{(0...T)})log[ \frac{p(x^{(0)}|x^{(1)})}{q(x^{(1)}|x^{(0)})}]-H_p(x^{(T)})$$边缘化无关变量，且由于$p(x^{(0)}|x^{(1)})=q(x^{(1)}|x^{(0)})\frac{\pi (x^{(0)})}{\pi (x^{(1)})}$则：$$K= \sum _{t=2}^{T}\int dx^{(0...T)}q(x^{(0...T)})log[ \frac{p(x^{(t-1)}|x^{(t)})}{q(x^{(t)}|x^{(t-1)})}]+\int dx^{(0,1)}q(x^{(0,1)})log[ \frac{q(x^{(1)}|x^{(0)})\pi(x^{(0)})}{q(x^{(1)}|x^{(0)})\pi(x^{(1)})}]-H_p(x^{(T)})$$由于设计前向过程，熵$H_p(x^{(t)})=H_p(x^{(T)})=\mathbb{E}_p(-log(\pi(x)))$则中项展开后消去：$$=\sum _{t=2}^{T}\int dx^{(0...T)}q(x^{(0...T)})log[ \frac{p(x^{(t-1)}|x^{(t)})}{q(x^{(t)}|x^{(t-1)})}]-H_p(x^{(T)})$$前向过程满足Markov性，即$q(x^{(t)}|x^{(t-1)})=q(x^{(t)}|x^{(t-1)},x^{(0)})$，则:$$=\sum _{t=2}^{T}\int dx^{(0...T)}q(x^{(0...T)})log[ \frac{p(x^{(t-1)}|x^{(t)})}{q(x^{(t)}|x^{(t-1)},x^{(0)})}]-H_p(x^{(T)})$$由贝叶斯规则$q(x^{(t)}|x^{(t-1)},x^{(0)})\frac{q(x^{(t-1)}|x^{(0)})}{q(x^{(t)}|x^{(0)})}=q(x^{(t-1)}|x^{(t)},x^{(0)})$：$$=\sum _{t=2}^{T}\int dx^{(0...T)}q(x^{(0...T)})log[ \frac{p(x^{(t-1)}|x^{(t)})}{q(x^{(t-1)}|x^{(t)},x^{(0)})}\frac{q(x^{(t-1)}|x^{(0)})}{q(x^{(t)}|x^{(0)})}]-H_p(x^{(T)})$$拆开,第二项写成条件熵:$$\begin{align*}\sum _{t=2}^{T}\int dx^{(0...T)}q(x^{(0...T)})log[\frac{q(x^{(t-1)}|x^{(0)})}{q(x^{(t)}|x^{(0)})}]\\
=\sum _{t=2}^{T}\int dx^{(0...T)}q(x^{(0...T)})log[{q(x^{(t-1)}|x^{(0)})}]
-\sum _{t=2}^{T}\int dx^{(0...T)}q(x^{(0...T)})log[{q(x^{(t)}|x^{(0)})}]\\
=\sum _{t=2}^{T}\int dx^{(0,t)}q(x^{(0)})q(x^{(t)}|x^{(0)})log[{q(x^{(t)}|x^{(0)})}]-\sum _{t=2}^{T}\int dx^{(0,t-1)}q(x^{(0)})q(x^{(t-1)}|x^{(0)})log[{q(x^{(t-1)}|x^{(0)})}]\\
=\sum_{t=2}^{T}[H_q(x^{(t)}|x^{(0)})-H_q(x^{(t-1)}|x^{(0)})]
\end{align*}$$则：$$=\sum _{t=2}^{T}\int dx^{(0...T)}q(x^{(0...T)})log[ \frac{p(x^{(t-1)}|x^{(t)})}{q(x^{(t-1)}|x^{(t)},x^{(0)})}]+\sum_{t=2}^{T}[H_q(x^{(t)}|x^{(0)})-H_q(x^{(t-1)}|x^{(0)})]-H_p(x^{(T)})$$合并：$$=\sum _{t=2}^{T}\int dx^{(0...T)}q(x^{(0...T)})log[ \frac{p(x^{(t-1)}|x^{(t)})}{q(x^{(t-1)}|x^{(t)},x^{(0)})}]+H_q(x^{(T)}|x^{(0)})-H_q(x^{(1)}|x^{(0)})-H_p(x^{(T)})$$由:$$D_{KL}(q(x^{(t-1)}|x^{(t)},x^{(0)})||p(x^{(t-1)}|x^{(t)}))=\int dx^{(t-1)}q(x^{(t-1)}|x^{(t)},x^{(0)})log(\frac{q(x^{(t-1)}|x^{(t)},x^{(0)})}{p(x^{(t-1)}|x^{(t)})})$$得到：
---
- **形式一:**$$\begin{align*}L\ge K\\
&=-\sum _{t=2}^{T}\int dx^{(0,t)}q(x^{(0,t)})D_{KL}(q(x^{(t-1)}|x^{(t)},x^{(0)})||p(x^{(t-1)}|x^{(t)}))\\
&+H_q(x^{(T)}|x^{(0)})-H_q(x^{(1)}|x^{(0)})-H_p(x^{(T)})
\end{align*}$$
---
# **Denoising Diffusion Probablilistic Models**

- 模型有：$$p(x^T)=N(x^T,\mathbb 0,I)$$定义:$$p_\theta(x^{t-1}|x^t):=N(x^{t-1};\mu_\theta(x^t,t),\Sigma_\theta(x^t,t))$$和:$$q(x^t|x^{t-1}):=N(x^{t};\sqrt{1-\beta_t}x^{t-1},\beta_tI)$$

- 原始公式：$$L\ge \int dx^{(0...T)}q(x^{(0...T)})\cdot log[ p(x^{(T)})\prod_{t=1}^{T}\frac{p(x^{(t-1)}|x^{(t)})}{q(x^{(t)}|x^{(t-1)})}]=K$$
- 令$V_{VLB}=-K$,由于：$$arg\max_\theta L=\mathbb{E}_{q(x^{(0)})}[\log p_\theta(x^{(0)})]\ge arg\max_\theta K\iff arg\min_\theta V_{VLB}=\mathbb{E}_{q(x^{(0)})}[-\log p_\theta(x^{(0)})]$$
- 则:$$\begin{align*}V_{VLB}&= -\int dx^{(0...T)}q(x^{(0...T)})\cdot \sum _{t=1}^{T}log[ \frac{p(x^{(t-1)}|x^{(t)})}{q(x^{(t)}|x^{(t-1)})}]-\int dx^{(T)}q(x^{(T)})log(p(x^{(T)}))\\
&=-\int dx^{(0...T)}q(x^{(0...T)})log[ \frac{p(x^{(0)}|x^{(1)})}{q(x^{(1)}|x^{(0)})}]-\sum _{t=2}^{T}\int dx^{(0...T)}q(x^{(0...T)})log[ \frac{p(x^{(t-1)}|x^{(t)})}{q(x^{(t)}|x^{(t-1)})}]\\&-\int dx^{(T)}q(x^{(T)})log(p(x^{(T)}))\\
&=-\int dx^{(0...T)}q(x^{(0...T)})log[ \frac{p(x^{(0)}|x^{(1)})}{q(x^{(1)}|x^{(0)})}]
\\&-[\sum _{t=2}^{T}\int dx^{(0...T)}q(x^{(0...T)})log[ \frac{p(x^{(t-1)}|x^{(t)})}{q(x^{(t-1)}|x^{(t)},x^{(0)})}]+H_q(x^{(T)}|x^{(0)})-H_q(x^{(1)}|x^{(0)})]\\&-\int dx^{(T)}q(x^{(T)})log(p(x^{(T)}))\\
\end{align*}$$
1. 对于上述原式:$$\sum _{t=2}^{T}\int dx^{(0...T)}q(x^{(0...T)})log[ \frac{p(x^{(t-1)}|x^{(t)})}{q(x^{(t)}|x^{(t-1)})}]=-\mathbb E_q[\sum _{t=2}^{T}D_{KL}(q(x^{(t-1)}|x^{(t)},x^{(0)})||p(x^{(t-1)}|x^{(t)}))]$$
2. 由于：$$\int dx^{(0...T)}q(x^{(0...T)})log[ \frac{p(x^{(0)}|x^{(1)})}{q(x^{(1)}|x^{(0)})}]=\mathbb E_{q(x^1)}[D_{KL}(q(x^{(0)}|x^{(1)}||p(x^{(0)}|x^{(1)}))]$$又：$$\begin{align*}
\int dx^{(0...T)}q(x^{(0...T)})log[ \frac{p(x^{(0)}|x^{(1)})}{q(x^{(1)}|x^{(0)})}]-H_q(x^{(1)}|x^{(0)})\\
=\int dx^{(0,1)}q(x^{(0,1)})log[ \frac{p(x^{(0)}|x^{(1)})}{q(x^{(1)}|x^{(0)})}]+\int dx^{(0,1)}q(x^{(0,1)})log[q(x^{(1)}|x^{(0)})]\\
= \int dx^{(0,1)}q(x^{(0,1)})log[ {p(x^{(0)}|x^{(1)})}]=\mathbb E_{q(x^0,x^1)}[log[p(x^{(0)}|x^{(1)})]]
\end{align*}$$
3. 剩余两项：$$\begin{align*}H_q(x^{(T)}|x^{(0)})+\int dx^{(T)}q(x^{(T)})log(p(x^{(T)}))\\=-\int dx^{(0,T)}q(x^{(0,T)})log(q(x^{(T)}|x^{(0)}))+\int dx^{(T)}q(x^{(T)})log(p(x^{(T)}))\\=-\int dx^{(0,T)}q(x^{(0,T)})log(q(x^{(T)}|x^{(0)}))+\int dx^{(0,T)}q(x^{(0,T)})log(p(x^{(T)}))\\=-\int dx^{(0,T)}q(x^{(0,T)})\cdot log\frac{q(x^{(T)}|x^{(0)})}{p(x^{(T)})}\\=-\int dx^{(0)}q(x^{(0)})D_{KL}[q(x^{(T)}|x^{(0)})||p(x^{(T)})]\\=-\mathbb E_q[D_{KL}[q(x^{(T)}|x^{(0)})||p(x^{(T)})]]
\end{align*}$$
---
- 也可以：根据$$\begin{align*}
\displaystyle
p(x^{(0)})&=\int dx^{(1:T)}
p(x^{(0:T)})
\frac{q(x^{(1:T)}|x^{(0)})}
{q(x^{(1:T)}|x^{(0)})}\\
&=\int dx^{(1:T)}
q(x^{(1:T)}|x^{(0)})
\frac{p(x^{(0:T)})}
{q(x^{(1:T)}|x^{(0)})}\\
&=\mathbb E_q[log\frac{p(x^{(0:T)})}
{q(x^{(1:T)}|x^{(0)})}]\\
L&=\mathbb E_q[-log\frac{p(x^{(0:T)})}
{q(x^{(1:T)}|x^{(0)})}]
\end{align*}$$展开计算。

则有：

---
- **形式二：**
$$V_{VLB}=\mathbb E_q[\underbrace{D_{KL}(q(x^{(T)}|x^{(0)})||p_\theta(x^{(T)}))}_{L_T}+\sum _{t=2}^{T}\underbrace{D_{KL}(q(x^{(t-1)}|x^{(t)},x^{(0)})||p(x^{(t-1)}|x^{(t)}))}_{L_{t-1}}-\underbrace{logp_\theta(x^{(0)}|x^{(1)})}_{L_0}]$$


---
- 当前向过程变量$\beta_t$通过重参数化学习，固定后，$L_T$为常数


---
- **优化目标**:
- 定义:
	- 在 DDPM / NTL（Non-equilibrium Thermodynamics）模型中，反向过程定义为：
$$p_\theta(x^{(t-1)}|x^{(t)}) = \mathcal{N}\big(x^{(t-1)}; \mu_\theta(x^{(t)}, t), \Sigma_\theta(t)\big)$$
	- $\theta$参数化了 均值函数 $\mu_\theta$（核心）
	- 协方差 $\Sigma_\theta$可以固定或参数化（早期论文通常固定）
- 理论目标$$\max_\theta\; \mathbb{E}_{q(x^{(0)})}[\log p_\theta(x^{(0)})]=L\ge K$$
- 变分下界(ELBO)：$$\max_\theta \; ​\mathbb E_{q(x^{(0)})}​[log p_\theta(x^{(0)})]⇔\max_\theta\;​K$$
- 实际优化目标:$$\max_\theta\; \mathbb{E}_{q(x^{(0)},x^{(1:T)})} \big[ \log p_\theta(x^{(0:T)}) - \log q(x^{(1:T)}|x^{(0)}) \big]$$
- 实际上，在训练中用 Monte Carlo 近似：
$$\int dx^{(0,t)} q(x^{(0,t)}) \, D_{\rm KL}(\cdot) \;\approx\; \frac{1}{N} \sum_{i=1}^N D_{\rm KL}(\cdot \mid x^{(0)}_i, x^{(t)}_i)$$
- 只训练 KL 项：条件熵项不依赖$\theta$:$$\min_\theta \sum_{t=2}^{T} D_{\rm KL}(q(x^{(t-1)}|x^{(t)},x^{(0)})\|p_\theta(x^{(t-1)}|x^{(t)})$$
- 进一步简化为噪声预测 MSE：$$\min_\theta \mathbb{E}_{x^{(0)}, \epsilon, t} \big[ \| \epsilon - \epsilon_\theta(x^{(t)}, t) \|^2 \big]$$