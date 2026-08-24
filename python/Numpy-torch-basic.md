### NumPy 与 PyTorch API 速查表
#### 1. 创建张量
| 操作 | NumPy API 示例 | PyTorch API 示例 |
| :--- | :--- | :--- |
| **从列表创建** | `np.array(object)`<br>输入列表，返回数组 | `torch.tensor(data)`<br>输入列表，返回张量；默认会推断数据类型 |
| **全零/全一矩阵** | `np.zeros(shape)` -> `0s, shape`<br>`np.ones(shape)` -> `1s, shape` | `torch.zeros(*size)` -> `0s, size`<br>`torch.ones(*size)` -> `1s, size` |
| **随机数 (正态)** | `np.random.randn(d0, d1)`<br>返回标准正态分布数据，形状为 `(d0, d1)` | `torch.randn(*size)`<br>返回标准正态分布数据，形状为 `size` |
| **序列** | `np.arange(start, stop, step)`<br>返回 `[start, stop)` 步长为 `step` 的序列 | `torch.arange(start, end, step)`<br>同左，注意参数名略有不同 |
| **未初始化** | `np.empty(shape)`<br>返回内存垃圾值，shape 同输入 | `torch.empty(*size)`<br>同左，常用于后续赋值预分配内存 |
| **单位矩阵** | `np.eye(N)`<br>返回 N×N 的单位矩阵 | `torch.eye(n)`<br>同左 |
#### 2. 形状操作
| 操作 | NumPy API 示例 | PyTorch API 示例 | 备注 |
| :--- | :--- | :--- | :--- |
| **查看形状** | `x.shape`<br>返回 tuple | `x.size()` 或 `x.shape`<br>返回 `torch.Size` (tuple子类) | |
| **重塑** | `np.reshape(a, newshape)`<br>返回新形状视图或副本 | `x.reshape(*shape)` 或 `x.view(*shape)` | **PyTorch**: `view` 要求内存连续，`reshape` 兼容性更好 |
| **展平** | `np.ravel(a)`<br>返回视图(尽量) | `x.flatten()`<br>返回连续副本 | `flatten` 更安全，`ravel` 更省内存 |
| **增加维度** | `np.newaxis` (切片用法)<br>如 `x[:, None]` | `x.unsqueeze(dim)`<br>在指定 `dim` 插入长度为1的维度 | PyTorch 函数式更直观 |
| **减少维度** | `np.squeeze(a)`<br>移除所有长度为1的维度 | `x.squeeze(dim)`<br>不填 `dim` 则移除所有1维；指定则移除特定维 | |
| **维度置换** | `np.transpose(a, axes)`<br>如 `(1, 0, 2)` | `x.permute(*dims)`<br>如 `(1, 0, 2)` | **注意**: PyTorch 用 `permute`，NumPy 用 `transpose` |
#### 3. 数学运算：聚合与归约
| 操作 | NumPy API 示例 | PyTorch API 示例 | 关键差异 |
| :--- | :--- | :--- | :--- |
| **求和** | `np.sum(a, axis=None)`<br>返回标量或降维数组 | `x.sum(dim=None)`<br>返回 0-dim Tensor | `axis` vs `dim` |
| **均值** | `np.mean(a, axis)` | `x.mean(dim)` | 同上 |
| **最大值** | `np.max(a, axis)`<br>仅返回最大值 | `x.max(dim)`<br>返回 `NamedTuple` (values, indices) | **重要**: PyTorch 默认返回索引，便于计算准确率 |
| **ArgMax** | `np.argmax(a, axis)`<br>返回索引 | `x.argmax(dim)`<br>返回索引 | 一致 |
| **保持维度** | `np.sum(a, axis, keepdims=True)`<br>输出 shape 保持原维度大小为 1 | `x.sum(dim, keepdim=True)`<br>同左 | **核心技巧**：设为 True 可避免广播错误 |
#### 4. 数学运算：逐元素
| 操作 | NumPy API 示例 | PyTorch API 示例 | 说明 |
| :--- | :--- | :--- | :--- |
| **指数** | `np.exp(x)` -> $e^x$ | `torch.exp(input)` -> $e^x$ | 常用于 Softmax |
| **对数** | `np.log(x)` -> $\ln(x)$ | `torch.log(input)` | 常用于 NLLLoss |
| **截断** | `np.clip(x, min, max)` | `torch.clamp(input, min, max)` | 将数值限制在 `[min, max]`，用于梯度裁剪或 ReLU 实现 |
#### 5. 线性代数
| 操作 | NumPy API 示例 | PyTorch API 示例 | 备注 |
| :--- | :--- | :--- | :--- |
| **矩阵乘法** | `np.dot(a, b)` 或 `a @ b` | `torch.mm(a, b)` 或 `a @ b` | `mm` 仅限 2D 矩阵 |
| **批量矩阵乘** | `np.matmul(a, b)`<br>支持广播 | `torch.bmm(a, b)`<br>仅限 3D (Batch, N, M) | `torch.matmul` 功能类似 `np.matmul` |

---
### 重点技巧详解与示例
#### 1. 广播机制
**原理**：两个张量进行运算时，如果形状不同，系统会自动尝试“扩展”较小的张量形状，使其与较大张量兼容。
**规则**：从尾部维度对齐，维度相等或其中一个为 1 即可广播。
**示例：批量向量归一化**
假设我们有一个 Batch 的数据 (32, 100)，我们想用一组权重 (100,) 进行加权求和。
```python
import torch
import numpy as np
# 场景：32个样本，每个样本100个特征
data = torch.randn(32, 100)  # shape: (32, 100)
weights = torch.randn(100)     # shape: (100,)
# ❌ 错误思维：显式复制32次权重 (浪费内存)
# weights_expanded = weights.expand(32, -1) 
# ✅ 正确做法：利用广播直接相乘
# (32, 100) * (100,) -> (32, 100) * (1, 100) -> (32, 100)
result = data * weights  # 无需修改代码，自动广播
# 常见陷阱：如果是列向量相加
# (32, 100) + (32, 1) -> OK (列广播)
# (32, 100) + (32,)   -> Error! (尾部对齐，(32,)被当做(1, 32)，不匹配(100,))
# 修复：vec.unsqueeze(1) 变为 (32, 1)
```
#### 2. Keepdim 机制
**原理**：在聚合运算（如 sum, max）时，如果不设置 `keepdim=True`，被操作的维度会消失，导致形状不匹配。设置 `True` 后，该维度保留为 1，便于后续广播。
**示例：计算 Softmax**
Softmax 公式需要减去均值，如果不保持维度，广播会失败。
```python
logits = torch.randn(4, 10)  # Batch=4, Classes=10
# 计算每个样本的最大值
# ❌ 不使用 keepdim
max_val = logits.max(dim=1).values  # shape: (4,)
# 尝试计算: logits - max_val 
# (4, 10) - (4,) -> Error! 形状不匹配
# ✅ 使用 keepdim=True
max_val_keep = logits.max(dim=1, keepdim=True).values # shape: (4, 1)
# 尝试计算: logits - max_val_keep
# (4, 10) - (4, 1) -> OK! 广播成功
result = logits - max_val_keep
```
#### 3. In-place 操作 (节省显存)
**原理**：PyTorch 中带下划线后缀 `_` 的方法（如 `add_`, `mul_`）是原地操作，直接修改原张量内存，不分配新内存。
**优点**：节省显存，减少拷贝开销。
**缺点**：会覆盖历史数据，可能导致计算图断裂（无法反向传播）。
**示例：梯度截断与参数更新**
```python
# 假设这是模型的梯度
grad = torch.tensor([1.5, 2.5, -0.5])
# 1. 普通操作 (创建新对象)
# clipped = torch.clamp(grad, -1, 1) 
# grad 变量本身未变，clipped 是新变量
# 2. In-place 操作 (节省内存)
# 直接修改 grad 变量，不产生新对象
torch.clamp_(grad, min=-1, max=1) 
print(grad) # 输出: tensor([ 1.,  1., -0.5])，原变量已改变
# ⚠️ 注意：在神经网络训练中，尽量避免对叶子节点使用 in-place 操作
# w = torch.tensor([1.0], requires_grad=True)
# w.add_(1) # 报错：RuntimeError: a leaf Variable that requires grad is being used in an in-place operation.
```
#### 4. View vs Reshape vs Permute (内存视图)
这是一个 PyTorch 特有的高频考点。
*   **View**: 共享内存，要求张量在内存中必须是**连续** 的。如果不连续会报错。
*   **Reshape**: 更灵活。如果内存连续，效果同 `view`；如果不连续，会自动拷贝一份新数据使其连续。
*   **Permute**: 仅仅修改步长，不改变底层数据顺序。操作后张量通常变得**不连续**。
**示例：View 报错与修复**
```python
x = torch.arange(12).view(3, 4) # 连续内存
print(x.is_contiguous()) # True
# 进行维度置换，内存数据顺序未变，但逻辑顺序变了
y = x.permute(1, 0)      # shape (4, 3)
print(y.is_contiguous()) # False (关键点！)
# ❌ 尝试使用 View
# y_view = y.view(-1)   # 报错: RuntimeError: view size is not compatible...
# ✅ 解决方案 1: 使用 Reshape (会复制内存，开销较大)
z_reshape = y.reshape(-1) # 成功，但复制了数据
# ✅ 解决方案 2: 先 contiguous() 再 view (显式复制)
z_cont = y.contiguous().view(-1)
```
