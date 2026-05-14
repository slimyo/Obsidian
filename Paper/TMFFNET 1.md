# 论文修订补丁清单

> **使用说明**：本文件仅列出**新增内容**与**精确插入位置**，原稿不删一字、不改一句。共 7 处插入，新增正文约 740 词；另含 5 个 LaTeX 公式可直接粘贴到 Word 公式编辑器（Word 公式工具支持 LaTeX 输入：`插入 → 公式 → 转换 → LaTeX 数学`，或在公式框内用 `Alt + =` 进入数学模式后输入）。
> 
> **修订前先做的一件事**：确认 MatchingNet [28] 的引用已从 _Chang & Chen (2018) "Pyramid Stereo Matching Network"_ 改为 _Vinyals et al. (2016) "Matching Networks for One Shot Learning", NeurIPS 29:3630–3638_。这是审稿人2明确点名的硬伤，必须修正。

---

## 补丁 1 — 引言增加"声学–振动物理同源性"一句

**回应**：编辑意见"limited connection to vibration and control"（与 JVC 期刊范围契合度问题）。

**插入位置**：Introduction 第 1 段末尾。原段以 "…have thus emerged as a critical source for condition monitoring." 结尾，紧接其后的下一段（"Recent studies have explored…"）之前。

**直接在该段末尾追加一句**（不另起新段）：

> From a physical standpoint, fault-induced acoustic emissions originate from structure-borne vibrations radiating through the surrounding fluid medium, and therefore inherit the modulation, harmonic, and impact characteristics of the source vibration; this intrinsic equivalence allows the direct application of vibration-domain analytical tools such as envelope spectrum and characteristic-frequency analysis.

---

## 补丁 2 — 引言贡献条目之后增加"新颖性澄清"一段

**回应**：编辑+审稿人2的最核心质疑"the methodological novelty is limited"和"the paper does not clearly explain what is fundamentally new compared with existing Transformer-based, ResNet-based, and prototypical-network-based few-shot fault diagnosis methods"。

**插入位置**：Introduction 中三条贡献（"…enhancing classification robustness under limited data conditions."）**之后**、章节组织段（"The paper is organized as follows…"）**之前**。

**新增一个独立段落**（IOPText 样式）：

> Unlike existing Transformer-, ResNet-, and prototypical-network-based few-shot fault diagnosis methods that emphasize either single-path temporal modeling or top-down / symmetric feature fusion borrowed from computer vision, the novelty of TMFFNet lies in two aspects. First, the dual-path backbone is explicitly matched to the dual-scale nature of acoustic fault signatures — namely, sub-millisecond local transients coupled with long-range periodic modulations — rather than being motivated solely by component-level performance. Second, the proposed BU-Guided Fusion reverses the conventional fusion direction so that low-level high-resolution features, where transient fault information is concentrated, guide the higher-level semantic representations rather than the reverse. To our knowledge, this bottom-up fusion direction is the first to be physically motivated and experimentally validated for machinery acoustic diagnosis.

---

## 补丁 3 — Methodology 开头增加"物理动机"一段（含 2 个公式）

**回应**：审稿人2 "lacks sufficient physical interpretation of the acoustic fault features"。

**插入位置**：Section 3 Methodology 第 2 段（"Figure 1 illustrates the overall architecture of TMFFNet…"，以"…final classification is performed via Euclidean distance to the class prototypes."结尾）**之后**、第 3.1 节 "TransBlock design" 标题**之前**。

**新增一个独立段落**（IOPText 样式）：

> From a physical standpoint, the acoustic signatures of pump and gear faults exhibit two distinct temporal scales. Low-level transient impacts — originating from local defects such as blade damage, cavitation bubble collapse, or gear-tooth pitting — appear as sub-millisecond impulses with broadband high-frequency energy. High-level periodic modulations — dictated by rotational kinematics — appear as the blade-passing frequency in centrifugal pumps and the gear-meshing frequency together with its shaft-frequency sidebands in gearboxes:

**紧接其后插入两个公式**（编号需与原稿连续，下面假设接 (4) 之后；请按实际重新编号）：

```latex
f_{\mathrm{BPF}} = N_{\mathrm{blade}} \cdot f_r,
```

```latex
f_{\mathrm{mesh}} = Z \cdot f_r, \qquad f_{\mathrm{mesh}} \pm k\, f_r,\; k = 1, 2, \ldots
```

**公式之后追加一句**：

> where $N_{\mathrm{blade}}$ is the number of pump blades, $Z$ is the number of gear teeth, and $f_r$ is the shaft rotation frequency. These two scales cannot be jointly captured by a single fixed-resolution representation, which directly motivates the dual-path backbone and multi-level feature fusion adopted in TMFFNet.

---

## 补丁 4 — BU-Guide Fusion 增加"为何自下而上"一段

**回应**：编辑要求清楚区分本方法与现有特征融合方法；审稿人2要求论证组合的本质创新。

**插入位置**：Section 3.4 BU-Guide Fusion 第 1 段（"To fuse the multi-level temporal features extracted by TResNetBlock, a bottom-up guided feature fusion module, termed BU-Guided Fusion, is designed…"）末尾。可直接追加到该段末，也可另起一段。**建议另起一段**以突出重要性：

> The choice of bottom-up direction is motivated by a physical observation: in machinery acoustic signals, the diagnostically discriminative content is concentrated in high-temporal-resolution, low-level transients (impulse shapes, sub-millisecond rise times, broadband energy distributions), whereas higher-level features obtained by repeated downsampling gain semantic abstraction at the cost of attenuating exactly these transients. Accordingly, low-level features are used as the gating controllers and high-level features as the signals being gated. This direction is the opposite of the conventional top-down fusion adopted in image segmentation, where high-level semantic features serve as the primary signal to be refined.

---

## 补丁 5 — Section 4（实验）新增"Few-shot Evaluation Protocol"子章节

**回应**：审稿人2 #2 "the precise episodic training/testing protocol is not adequately described" + 50/100/500/1000 含义不清。这是最关键的补充。

**插入位置**：Section 4.1 Dataset **之后**、Section 4.2 Model parameters design **之前**。

**新增一个 IOPH2 二级标题**：

> ### 4.2 Few-shot evaluation protocol

（原来的 4.2 Model parameters design 顺延为 4.3，后续节号依次顺延一位。）

**该子节正文**（5 段，IOPText 样式；每段以加粗的小标题起头）：

> To ensure rigorous and reproducible evaluation, the few-shot setting follows the standard episodic meta-learning protocol with the following explicit specifications.

> **Class split.** For each dataset, the classes are partitioned into disjoint meta-training and meta-testing subsets, so that all classes encountered at test time are entirely unseen during meta-training. The specific class assignments for CPD and MGD are reported in Table 1.

> **Episode construction.** Each training/testing episode samples $N$ classes from the corresponding split. From each sampled class, $K$ frames form the support set $\mathcal{S}$ and $Q$ additional frames form the query set $\mathcal{Q}$, all drawn without replacement. Throughout this paper we use $N = 5$, $K \in {1, 5}$, and $Q = 15$.

> **Clarification of sample size.** The labels "50 / 100 / 500 / 1000 samples" in the results refer to the total number of available frames per meta-test class from which support and query sets are drawn during evaluation; the model itself observes only $K = 1$ or $K = 5$ support frames per class in each episode regardless of the pool size. The pool size therefore controls the diversity of sampled episodes rather than the support-set size.

> **Testing protocol.** At meta-test time, 200 independent episodes are sampled from the meta-test classes. For each episode, accuracy, precision, recall, and F1-score are computed over the $N \times Q = 75$ queries. Mean values across the 200 episodes are reported as the final metrics.

> ⚠️ **以上含两处变量取值需您核对并填入实际值**：(i) 各数据集的 meta-train / meta-test 类别划分（建议直接列入 Table 1 添加一栏"split"）；(ii) "50/100/500/1000 samples" 的含义说明 — 上面按"每类可用帧数池"解释；若您实际指其它含义（如训练集总样本数），**必须改写第 3 段**，这是审稿人最大困惑点。

---

## 补丁 6 — Section 4 比较实验段后增加"公平比较+基线说明"两段

**回应**：审稿人1 #5（comparison models 缺架构说明）+ 审稿人2 #4（fair-comparison 公平性存疑）。

**插入位置**：Section 4.5（原 4.5 Comparison with prevalent models，按补丁 5 顺延应为 4.6）正文（以 "…demonstrating its effectiveness and robustness in few-shot scenarios." 结尾）**之后**、Table 6 之前；或紧贴 Table 6 之后皆可。**建议放在该段之后、Table 6 题注之前**：

**新增第 1 段（公平比较）**：

> **Fair-comparison protocol.** To ensure that the observed differences reflect methodological merit rather than implementation artifacts, all baselines were reimplemented in the same PyTorch codebase using the same 35-dimensional multi-domain feature inputs, identical meta-train / meta-test class splits, the same training budget, and a common hyperparameter search space (learning rate and batch size tuned over the same grid as TMFFNet). The same 200-episode testing protocol described in §4.2 is applied to every baseline.

**新增第 2 段（基线说明）**：

> **Baseline descriptions.** _LSTM-ProtoNet_ [27] combines a two-layer LSTM encoder with a prototypical-network head, representing the recurrent-embedding metric-based baseline. _TransBlock_ [27] uses a Transformer encoder identical to our TransBlock branch paired with a prototype head, isolating the long-range modeling contribution. _MatchingNet_ [28] employs attention-based comparison of query and support embeddings without explicit prototypes. _RelationNet_ [29] learns a non-linear comparator via two CNN modules — one for embedding and one for relation scoring. _SiameseWDCNN_ [30] uses a wide-kernel deep CNN within a Siamese framework, representing the CNN-based metric-learning baseline. _ETMM_ [4] applies multi-scale Transformer with Mahalanobis distance, and _QSformer_ [31] couples query-support attention with prototype matching; both are recent Transformer-based few-shot diagnosis methods. All baselines share the same input feature pipeline as TMFFNet to ensure comparability.

---

## 补丁 7 — Conclusions 末尾追加"局限性与未来工作"

**回应**：审稿人2 关于实验范围有限（仅 2 个数据集、缺乏跨工况/跨域/噪声实验）的批评。主动声明局限比让 reviewer 挑出来要好得多。

**插入位置**：Conclusions 唯一一段（以 "…validating its effectiveness and generalization capability." 结尾）**之后**、Data availability statement 标题**之前**。

**新增一个独立段落**（IOPText 样式，以加粗小标题起头）：

> **Limitations and future work.** Three limitations of the present study should be acknowledged and will be addressed in future work: (i) the CPD was collected under a single operating condition, leaving cross-condition generalization unverified; (ii) the MGD experiments use only the source-domain subset of DCASE 2021 Task 2, without exploiting the inherent source-to-target domain shift of that benchmark; and (iii) noise robustness under low signal-to-noise-ratio field conditions has not been characterized. Extending TMFFNet to cross-condition, cross-domain, and noise-robust scenarios, and transferring the bottom-up guided fusion mechanism to vibration signals, are promising directions for future research.

---

## 涉及到的 LaTeX 公式汇总（便于一次粘贴）

公式 (a) 叶片通过频率：

```latex
f_{\mathrm{BPF}} = N_{\mathrm{blade}} \cdot f_r
```

公式 (b) 齿轮啮合频率与边带：

```latex
f_{\mathrm{mesh}} = Z \cdot f_r, \qquad f_{\mathrm{mesh}} \pm k\, f_r,\; k = 1, 2, \ldots
```

文中行内变量（补丁 5 用到）：

- 支持集 $\mathcal{S}$ — `\mathcal{S}`
- 查询集 $\mathcal{Q}$ — `\mathcal{Q}`
- N-way K-shot Q-query — `$N = 5$, $K \in \{1, 5\}$, $Q = 15$`
- 75 个查询 — `$N \times Q = 75$`

**Word 中插入 LaTeX 公式的步骤**：

1. 光标定位 → `插入` 选项卡 → `公式`（或直接按 `Alt + =`）
2. 进入公式框后，选 `转换` → `LaTeX 数学` 模式
3. 粘贴上面的 LaTeX 代码（不含外侧 ` ``` `）
4. 按 `Enter` 或点击公式框外完成转换

---

## 修订量统计

|补丁|新增正文（词）|新增公式|涉及章节|
|---|---|---|---|
|1. 声学–振动同源性|50|—|Introduction|
|2. 新颖性澄清|110|—|Introduction|
|3. 物理动机|95|2|Methodology 开头|
|4. 自下而上理由|90|—|Section 3.4|
|5. Few-shot 协议|220|行内变量|Section 4 新增子节|
|6. 公平比较 + 基线说明|200|—|Section 4 比较实验段后|
|7. 局限性与未来工作|90|—|Conclusions 末尾|
|**合计**|**约 855 词**|**2 个显示公式**|—|

原稿正文（含摘要）约 5800 词，新增量占 14.7%。所有插入点均位于段落或子节边界，**不删除、不改写、不重组原文任何内容**。

---

## 还有一件可选的事

如果您愿意把 Table 1（Model parameters）扩充一点点（加 4–5 行：optimizer、learning rate、weight decay、scheduler、batch size、total episodes），就能完整回应 Reviewer 1 第 2 条 "hyperparameter table"。这是真的"加 5 行字"的小事，不必单独算一处补丁。

---

如果您希望我接下来：

1. **把这 7 处补丁直接以"Word 修订/批注"格式插入您的 docx**（您打开后能看到红色添加痕迹，逐条接受/拒绝）；
2. **起草 Response Letter 的逐条回复模板**（针对 AE + Reviewer 1 + Reviewer 2 的每一条意见，按"审稿意见 → 我们的回复 → 论文中对应修改位置"三段式）；
3. **检查您改正的 MatchingNet 引用是否完整正确**（包括正文 [28] 出现的所有位置）；

任选一项告诉我。