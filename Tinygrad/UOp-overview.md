# 一、UOp 图系统（表示层）

## 1. 核心定义

UOp（Unified Operation）是 tinygrad 的计算节点，所有计算都表示为一个 **有向无环图（DAG）**：

- 节点：`UOp`
- 边：数据依赖（`src`）

每个 UOp 由以下字段唯一确定：

- `op`：操作类型（来自 `Ops`）
- `dtype`：数据类型
- `src`：输入节点（tuple）
- `arg`：附加参数（shape / axis / 常量值等）
- `tag`（可选）：编译阶段标记

👉 本质：

> **计算 = UOp DAG**

---

## 2. 结构特性

### （1）不可变 + 可哈希

- 所有 UOp 是不可变对象
- 可作为 key 使用（支持缓存与去重）

### （2）DAG 结构

- 通过 `src` 引用形成依赖图
- 不允许环

### （3）拓扑序支持

- `toposort()` 保证：
    - 先计算依赖
    - 后计算消费者

---

## 3. 结构共享（核心优化）

UOp 使用元类实现**全局去重**：

(op, dtype, src, arg, tag) → 唯一实例

### 关键机制

- `UOpMetaClass.ucache`
- 弱引用缓存（`weakref`）

### 效果

- 相同子图只存在一份
- 可直接用 `is` 判断相等
- 自动实现：
    - 公共子表达式消除（CSE）
    - 内存优化
    - 加速图重写

---

## 4. 递归属性系统

部分属性通过递归定义并缓存：

- `_shape`
- `vmin / vmax`
- `ranges`

👉 特点：

- 按需计算
- 自动缓存
- 避免重复递归

---

## 5. UOp 的本质

可以把 UOp 图理解为：

|层级|含义|
|---|---|
|高层|Tensor 计算表达式|
|中层|符号计算图（代数表达）|
|底层|硬件执行指令（最终目标）|

---

# 二、UOp 构建机制（Construction）

## 1. 构建入口

所有 Tensor 操作最终都会转化为 UOp：

a + b → UOp(Ops.ADD, dtype, (a, b))

👉 注意：

- **不会立即执行**
- 只构建图（lazy）

---

## 2. 构建流程

---

## 3. 构建特性

### （1）结构相等 = 实例相同

- 自动 dedup

### （2）弱引用管理

- 无引用自动释放
- 防止内存泄漏

### （3）即时简化（部分）

- 常量折叠
- 单位元消除

---

## 4. 核心哲学

> **构建阶段就开始优化（而不是等编译）**

---

# 三、图重写系统（Rewriting）

UOp 图不会直接执行，而是通过**模式匹配 + 重写规则**不断优化。

---

## 1. 三大核心组件

### （1）UPat（模式语言）

用于描述子图结构：

UPat(Ops.ADD, src=(UPat(Ops.CONST), UPat(Ops.CONST)))

---

### （2）PatternMatcher

- 管理规则集合
- 执行匹配与替换

---

### （3）graph_rewrite（执行引擎）

核心流程：

1. 拓扑排序
2. 遍历节点
3. 匹配规则
4. 替换节点
5. 迭代直到稳定

---

## 2. 重写策略

- **bottom-up**（常用）
- **top-down**
- **fixpoint（直到不再变化）**

---

## 3. 重写类型

### （1）符号简化

- `x + 0 → x`
- `x * 1 → x`
- `(x*c1)+(x*c2) → x*(c1+c2)`

---

### （2）代数优化

- 常量折叠
- 因式分解
- 比较简化

---

### （3）图结构优化

- 消除冗余节点
- 融合操作
- 调整依赖结构

---

## 4. 核心思想

> **优化 = 图等价变换**

---

# 四、图转换与编译流程（Pipeline）

tinygrad 没有多层 IR：

> **从头到尾只有 UOp 一种表示**

---

## 整体流程

Tensor → UOp DAG → Rewrite → Rangeify → Kernel → Codegen → 执行

---

# 五、UOp 编译主流程

## （1）前端：构建 DAG

- Tensor 操作 → UOp
- 惰性执行
- 图不断增长

---

## （2）触发编译：Big Sink

- 插入统一终点节点
- 确保覆盖整个图

---

## （3）Schedule 规范化

- 去除具体值差异
- 提升缓存命中

---

# 六、Rangeify（核心阶段）

（保留你的图位置）

![[tinygrad_uop_pipeline_overview.svg|1000]]

---

## 1. 核心目标

> **决定：哪些操作可以融合，哪些必须拆开**

---

## 2. Shape → Range

- Shape：描述“是什么”
- Range：描述“怎么执行”

---

## 3. Range 与 Index

- Range：循环变量
- Index：内存访问表达式

---

## 4. 核心流程（精简版）

### ① 打标签（Tagging）

- DFS 标记节点

### ② 构建 Realize Map

- 标记必须 materialize 的节点

### ③ Range 传播（核心）

- 从输出向输入传播

规则：

- Permute → 交换轴
- Expand → 广播维度为 0
- Shrink → 加偏移
- Flip → 反向索引

---

### ④ 符号简化

- 优化索引表达式

---

### ⑤ Buffer 决策（融合边界）

- 决定是否插入中间 buffer

---

### ⑥ Bufferize

分解为：

- BUFFER
- RANGE
- INDEX
- STORE
- END

---

### ⑦ Kernel 切分

- 生成独立 kernel
- 构造 CALL UOp

---

### ⑧ WAR 依赖

- 防止读写冲突

---

### ⑨ Result Map

- 原 UOp → kernel 映射

---

# 七、Lowering 与执行

## （1）Kernel DAG → 线性执行

- 拓扑排序
- 得到 ExecItem 列表

---

## （2）Lowering

- AST → 线性 UOp
- BEAM search 优化循环结构

---

## （3）Renderer

生成目标代码：

- CUDA
- OpenCL
- Metal
- C

---

## （4）Compiler

- 编译为二进制

---

## （5）执行

- 在设备上运行 kernel

---

# 八、整体总结（关键一句话）

> **tinygrad 的本质是：用一张 UOp 图统一表达计算，并通过持续的图重写与 range 化，将高层张量操作逐步转化为底层硬件循环与指令。**

---

# 九、最终结构记忆（强烈建议记住）

UOp（表示）  
  ↓  
Construction（构建）  
  ↓  
Rewrite（优化）  
  ↓  
Rangeify（并行化 & 融合）  
  ↓  
Kernel（计算单元）  
  ↓  
Lowering（线性化）  
  ↓  
Codegen（代码生成）  
  ↓  
Execution（执行）

---
